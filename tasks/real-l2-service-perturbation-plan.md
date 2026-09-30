# Real-L2 Matrix Service Perturbation Improvement Plan

## 1. Background

The CUTE matrix memory path targets eight physical L2 banks and expects one
64-byte response from every bank per cycle, for a peak read bandwidth of
512 B/cycle. AB requests use the identity bank mapping:

```text
physical L2 bank = request channel = MatrixReg destination bank
bank = (address >> 6) & 0x7
```

The response path is:

```text
CUTE request
  -> LocalMMU / Cute2TL
  -> L2 SinkA / RequestBuffer / RequestArb
  -> Directory / MainPipe / MSHR
  -> GrantBuffer / CustomL1Hint / matrixDataOut
  -> ResponseChannelBridge / Omega
  -> loader response handling
  -> MatrixReg writeback
```

The workload warms the cache before entering the matrix trace window. The
cache is large enough to retain the working set, and the run was intended to
measure an all-hit, interference-free real-L2 path.

The diagnostic implementation added two complementary schemas:

- `cache-matrix-trace-v1`: window lifecycle, matrix misses, replacements, and
  non-matrix traffic;
- `cache-service-trace-v1`: RequestArb, directory result, MainPipe response,
  GrantBuffer, hint queue, L2 output masks, ResponseBridge/Omega masks, and
  MatrixReg write masks.

Primary evidence is under:

```text
build.0901_cache_disturb/zames_tiling_check_debug.log
build.0901_cache_disturb/0901_cache_disturb.zames_tiling_check_debug_mem_report/
build.0901_cache_disturb/coupled_l2_8bank_perf.json
```

## 2. Observed Behavior

### 2.1 Overall performance

The run contains 10,240 complete memory traces. All 6,144 reads complete in
1,717 active cycles, yielding 229.01 B/cycle.

```text
CRead L2 service:  avg 34.4, p90 143, p99 159, max 169 cycles
CWrite L2 service: avg 31.4, p90 142, p99 158, max 209 cycles
```

The first large batch sustains eight sends and eight completions from cycle
527394 through 527521. At cycle 527522, eight AReads are sent but only six
AReads complete.

### 2.2 Cache residency and external interference are not the cause

The trace window is complete and reports:

```text
matrix misses       = 0
valid replacements  = 0
non-matrix requests = 0
```

Therefore the slowdown is not caused by capacity/conflict misses, eviction of
the warmed data, or new L1/non-matrix traffic during matrix execution.

### 2.3 Main cause: coherent hits that require probes

All 12,022 directory observations hit, but 1,782 observations report:

```text
hit=1, has_clients=1, need_probe=1, need_mshr=1
```

Matching every directory event to its request gives:

| Request | Count | Requests requiring probe | Normal task-to-Grant | Probe task-to-Grant |
| --- | ---: | ---: | ---: | ---: |
| ARead | 1,024 | 0 | avg 4.9 cycles | - |
| BRead | 1,024 | 0 | avg 5.7 cycles | - |
| CRead | 4,096 | 891 | avg 4.6 cycles | avg 125.0, max 168 |
| CWrite | 4,096 | 891 | avg 5.1 cycles | avg 116.9, max 207 |

The 891 CRead and 891 CWrite probe requests access exactly the same 891 unique
addresses. The directory therefore retains upper-level client ownership for a
specific portion of the warmed C working set. A directory hit is not an
L2-only hit: the real L2 must probe the client and allocate MSHR state before
returning data.

The current warmup uses `AcquireBlock`. It establishes L2 residency but does
not guarantee that the line has no upper client. Thus:

```text
no runtime L1 request != no residual L1 coherence ownership
```

### 2.4 Secondary cause: CLoad Omega backpressure

CLoad response statistics are:

```text
responses offered              4,561
responses accepted             4,096
input stalled response-cycles    465
Omega blocked cycles             241
Omega blocked response-cycles    747
```

At the L2 boundary there are also exactly 465 valid-but-not-fired matrix
response-cycles across 201 cycles. Those 201 cycles coincide with CLoad bridge
input stalls and Omega blocking. The causal chain is:

```text
CLoad Omega conflict
  -> ResponseBridge input backpressure
  -> matrixDataOut.ready low
  -> GrantBuffer dequeue stalls
  -> additional response skew
```

AML and BML have zero Omega blocking. CStore has 390 blocked cycles, but its
bridge buffering absorbs them: all 4,096 offered responses are accepted, so it
does not propagate equivalent input backpressure into L2.

### 2.5 The first 8-to-6 event is not a MatrixReg loss

The masks form a one-cycle pipeline:

```text
527520 L2 matrix output fire_mask = 0x77
527521 AML bank_fire_mask         = 0x77
527522 AB MatrixReg write_mask    = 0x77

527522 AML bank_fire_mask         = 0xff
527523 AB MatrixReg write_mask    = 0xff
```

MatrixReg therefore writes exactly what the loader accepted one cycle earlier.
The missing banks at the first event are banks 3 and 7. Their requests entered
SinkA at 527515 but were admitted by RequestArb only at 527521 and 527520,
respectively; the other six requests entered RequestArb at 527515. At cycle
527520, the bank 3/7 GrantBuffers are empty rather than backpressured.

This first deficit is RequestBuffer/admission skew inside L2, not response loss
in Omega or MatrixReg.

### 2.6 Other observations

There are 10,238 stalled A-request-cycles:

```text
block_A/internal resource block  6,052
directory not ready              2,913
MSHR task priority               1,273
```

These are consistent with probe/MSHR work occupying shared slice resources and
propagating backpressure toward RequestBuffer. Current `reason=3` logging does
not distinguish MSHRCtl, MainPipe, GrantBuffer, TXDAT, and TXRSP blockers, so
the precise composition of those 6,052 cycles remains unresolved.

`CustomL1Hint` is not the direct cause of the observed 465 response stalls:

- matrix hint enqueue stalls: 0;
- `grantBufValid` blocks: 0;
- the 465 GrantBuffer stalls align with CLoad bridge/Omega backpressure.

## 3. Goals and Acceptance Criteria

The implementation should establish a real-L2 operating mode in which warmed
matrix data is served without residual upper-client probes and then quantify
the remaining Omega/admission limitations.

Success criteria:

1. The matrix trace window remains at zero misses, zero valid replacements,
   and zero non-matrix requests.
2. Steady matrix requests report `has_clients=0`, `need_probe=0`, and
   `need_mshr=0`, except for explicitly documented unavoidable traffic.
3. CRead/CWrite task-to-Grant latency loses the 140-200 cycle tail and
   approaches the current 4-7 cycle normal-hit distribution.
4. RequestArb full-width admission and L2 matrix output full-width cycles
   increase materially; every remaining deficit has a classified blocker.
5. ResponseBridge/Omega and MatrixReg counts remain exact: no response loss,
   duplication, or incorrect bank destination.
6. Functional output still passes the workload's complete element comparison
   and reaches `HIT GOOD TRAP`.
7. The read-bandwidth report is regenerated with the same methodology and
   compared against 229.01 B/cycle and the 512 B/cycle ceiling.

## 4. Implementation Plan

### Phase A: Prove and remove residual client ownership

- [ ] **A1. Identify ownership creation.** Trace the warmup path from CPU/L1
  `AcquireBlock` through L2 metadata updates. Confirm which of the C-region
  lines retain client bits and why only 891 of 4,096 CRead/CWrite requests need
  probes.
- [ ] **A2. Add an ownership-state summary.** At the end of warmup or the start
  of the trace window, report line counts by L2 state and client mask for the
  matrix address range. This is diagnostic only and must not affect cache
  control.
- [ ] **A3. Implement a clean warmup handoff.** Prefer an architectural or
  existing coherence operation that writes back/releases the warmed lines from
  L1 while retaining them in L2. The resulting L2 metadata must be valid with
  no upper client. Do not directly mutate directory metadata from debug logic.
- [ ] **A4. Evaluate an L2-side warmup alternative.** If the existing software
  warmup cannot relinquish ownership reliably, use an existing non-allocating
  or L2-directed mechanism to populate L2 without leaving data resident in L1.
  Avoid introducing a production-only cache bypass solely for the benchmark.
- [ ] **A5. Validate ownership cleanup.** Re-run the workload and require zero
  `need_probe`/`need_mshr` observations for the intended steady matrix range.

Recommended decision: implement A3 first because it preserves normal coherent
cache behavior and measures the production data path. Use A4 only if no valid
release/clean sequence can retain the lines in L2.

### Phase B: Classify and reduce L2 admission skew

- [ ] **B1. Split RequestArb reason 3.** Log independent blocker bits for
  MSHRCtl, MainPipe, GrantBuffer, TXDAT, TXRSP, same-set protection, and request
  priority. Preserve the compact reason code but do not use it in place of raw
  bits.
- [ ] **B2. Add RequestBuffer lifecycle events.** For each matrix request,
  record enqueue, selected, canceled/reselected, output valid, output fire,
  same-set block, MSHR-capacity throttle, and wait-cycle count with the original
  trace ID.
- [ ] **B3. Reproduce the bank 3/7 first deficit.** Explain why traces 1028 and
  1032 wait five to six cycles while the other six requests from the same batch
  issue immediately. Determine whether the cause is stale pipe status,
  same-set logic, queue arbitration, or another per-slice task.
- [ ] **B4. Fix only proven avoidable skew.** If the wait is caused by overly
  conservative status accounting or incorrect matrix-path capacity tracking,
  narrow that blocker with assertions proving GrantBuffer/MSHR safety. Do not
  bypass genuine coherence or queue-capacity constraints.
- [ ] **B5. Add a full-batch invariant test.** With eight independent hit lines,
  no client ownership, and all downstream ready, eight request channels must
  enter RequestArb and produce eight responses without bank-specific skew.

### Phase C: Reduce CLoad Omega backpressure

- [ ] **C1. Separate supply skew from routing conflict.** For every cycle,
  distinguish `L2 valid_mask != 0xff` from `valid_mask=0xff` with a partial
  ready/fire mask. The former is L2 supply skew; the latter is downstream
  routing/backpressure.
- [ ] **C2. Analyze conflict patterns.** Aggregate CLoad `omega_blocked_mask`
  and correlate it with source IDs, destination banks, response age, and batch
  membership. Confirm whether conflicts arise from responses of different
  batches becoming interleaved after variable L2 latency.
- [ ] **C3. Preserve batch alignment after L2.** Evaluate a bounded response
  resequencing or batch-aware admission policy before Omega so that one
  conflict-free eight-bank batch is presented together. Reuse the existing
  source/batch identity; do not introduce an unbounded reorder buffer.
- [ ] **C4. Compare scheduling alternatives.** Measure at least:
  existing Omega arbitration, oldest conflict-free selection, and batch-aware
  selection. Report area/state cost, added latency, blocked response-cycles,
  and sustained bandwidth.
- [ ] **C5. Maintain lossless behavior.** Assert stable payload under stall,
  exact one-time delivery, correct destination bank, and no cross-batch source
  confusion.

Phase C should be evaluated after Phase A. Removing probe-induced latency skew
may naturally restore aligned response batches and substantially reduce Omega
conflicts without adding response hardware.

### Phase D: Correct the analysis tools

- [ ] **D1. Fix MatrixReg anomaly detection.** A non-`0xff` write mask is not
  inherently anomalous. Compare a MatrixReg write mask against the matching
  loader acceptance mask at the known one-cycle latency and matching matrix
  register ID.
- [ ] **D2. Report the first causal deficit.** The parser should walk:

  ```text
  request batch
    -> RequestBuffer/RequestArb
    -> directory/MainPipe
    -> GrantBuffer/L2 output
    -> bridge input
    -> Omega output
    -> loader acceptance
    -> MatrixReg writeback
  ```

  It must distinguish no-valid supply gaps from valid-but-not-ready stalls.
- [ ] **D3. Add request-to-directory matching.** Handle source/trace ID reuse by
  correlating ID with lifecycle interval, bank, and address. Report probe and
  non-probe latency distributions separately.
- [ ] **D4. Add regression fixtures.** Include the 527520/527522 pipeline example,
  a legal partial MatrixReg write, a probe/MSHR long-latency request, and a true
  Omega-backpressure cycle.
- [ ] **D5. Update the research-agent guide.** Reflect the corrected one-cycle
  loader-to-MatrixReg relationship and the distinction between an L2 hit and
  an L2-only hit.

## 5. Validation Matrix

| Scenario | Expected evidence |
| --- | --- |
| Current 0901 baseline | 0 misses/replacements/non-matrix; 1,782 probe observations; 229.01 B/cycle |
| Ownership-clean warmup | 0 misses; 0 client/probe/MSHR matrix hits; no long coherence tail |
| Eight independent hits | RequestArb and L2 output masks remain `0xff` when downstream is ready |
| Forced RequestBuffer block | Exact raw blocker and trace wait interval are reported |
| Forced Omega conflict | L2 `valid=0xff`, partial ready/fire, stable response until acceptance |
| Partial legal writeback | Parser matches prior loader mask and does not flag response loss |
| Full workload | 10,240 complete traces, correct result, closed trace window, summary reconciliation |

Required commands include targeted Scala tests, parser unit tests,
`make comp`, `mill -i xiangshan.checkFormat`, and a complete
`make sim-verilog CONFIG=DefaultMatrixConfig`. The final performance run must
use the same workload, trace window, and bandwidth accounting as the 0901
baseline.

## 6. Deliverables

- RTL/software change that leaves warmed matrix lines in L2 with no upper
  client ownership.
- Expanded RequestBuffer/RequestArb blocker diagnostics.
- Updated service-trace parser and regression fixtures.
- Before/after latency distributions split by request kind and probe status.
- Before/after CLoad Omega blocked cycles and response-cycles.
- Before/after overall and per-batch read bandwidth.
- Updated analysis document containing direct evidence, inferred causes,
  excluded causes, and remaining evidence gaps.

## 7. Constraints

- Diagnostic signals must remain side-band and must not drive cache ready,
  arbitration, replacement, or coherence behavior.
- Do not treat directory hit as proof of a short L2-only service path.
- Do not fix the benchmark by silently bypassing coherence checks.
- Preserve the identity bank mapping and the existing Omega functional
  contract unless a separately reviewed architecture change requires otherwise.
- Preserve all unrelated staged and unstaged user changes in the root and both
  submodules.

The RequestBuffer-specific logging contract and post-processing procedure are
defined in `docs/requestbuffer-service-log-analysis-guide.md`.
