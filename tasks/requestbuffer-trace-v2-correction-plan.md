# RequestBuffer Trace v2 Correction Plan

## 1. Purpose

Revise the RequestBuffer diagnostic logging and parser added for the 0903
`hbl2_matrix_no_probe_reqbuf_profile` run. The revision must make every buffered
matrix request lifecycle unambiguous without changing RequestBuffer behavior,
timing decisions, arbitration, or cache protocol state.

The revised trace must answer, for every request delayed between SinkA and
RequestArb:

```text
Why was the request buffered?
Which condition kept it waiting each cycle?
Was it selected and then canceled?
When did it actually fire to RequestArb?
Did the same trace/source ID get reused for another request?
```

The companion parser must reconstruct these lifetimes, classify their first
and dominant blockers, and correlate a delayed bank with L2/Omega masks.

## 2. 0903 Findings That Motivate the Revision

The existing MEM workflow confirms that adding RequestBuffer logs did not
change workload behavior:

```text
10,240 / 10,240 MEM traces complete
CRead L2 service avg/p90/p99/max = 8.1 / 20 / 29 / 41 cycles
CWrite L2 service avg/p90/max    = 7.1 / 8 / 9 cycles
first short completion cycle     = 527522
first short send cycle           = 527577
```

The current RequestBuffer event counts are:

```text
RB-OUTPUT = 12,459
RB-ALLOC  = 4,271
RB-FREE   = 6,463
RB-WAIT   = 7
```

Matching current `RB-ALLOC` records to the first subsequent compatible
`RB-FREE` shows that all 4,271 allocated requests eventually leave the buffer.
The parser nevertheless reports `closed=0` because it waits for an event named
`RB-OUTPUT-FIRE`, which the RTL never emits.

The first two delayed AReads are directly visible:

```text
cycle 527515, trace 769:  mainpipe_block=1, wait_mp=4, set=52
cycle 527515, trace 1793: mainpipe_block=1, wait_mp=4, set=56
```

Their wait masks shift down and they leave at cycles 527521 and 527520. These
events support the earlier same-bank/same-set matrix-ordering conclusion.

However, the following trace defects prevent reliable full-run attribution:

1. `RB-ALLOC` reads `entry.rdy`, `entry.waitMP`, and `entry.waitMS` in the same
   clock block that assigns them. Printed register values may describe the old
   occupant/state while blocker fields describe the incoming request.
2. `chosen_cancel` is printed directly from `cancel`. When `chosenQValid=0`,
   `cancel` can be true due to an empty `Mux1H`; this is not a real cancellation.
3. `RB-FREE` is emitted whenever `chosenQ.io.deq.fire && y`, even when `cancel`
   is true, while the actual entry is cleared only when `!cancel`. This creates
   duplicate or false free records.
4. The parser groups by `(window, trace_id, entry)`. Matrix trace IDs and entry
   indices are reused, so separate lifetimes collapse into one group.
5. `RB-WAIT` is emitted only for `!e.rdy`. It does not explain time spent ready
   but waiting for issueArb/chosenQ/downstream admission.
6. The current trace does not emit the documented `RB-INPUT`, `RB-CHOSEN`,
   `RB-CHOSEN-CANCEL`, `RB-OUTPUT-FIRE`, or `RB-CYCLE` events.
7. Records omit explicit `bank`, `window`, buffer occupancy masks, `canFlow`,
   `doFlow`, `full`, and the separate same-set contributors `s1A_Block` and
   `s1B_Block`.

## 3. Non-Interference Requirements

- [ ] All new logic is observation-only. No trace-derived signal may drive
  `io.in.ready`, entry `rdy/valid`, issueArb, chosenQ, `cancel`, `io.out`, MSHR
  state, or MainPipe state.
- [ ] Compute printed fields with `Wire` aliases of existing combinational
  expressions or copies of already-computed state. Do not insert registers in
  a functional path.
- [ ] A diagnostic epoch counter may update only under existing alloc/free
  events and may be used only by `printf`/trace output.
- [ ] Keep trace logic behind the existing cache trace enable/build option and
  active window.
- [ ] Preserve disabled-trace generated logic and all externally visible
  handshakes.
- [ ] Do not change RequestBuffer ordering rules as part of this task. Any
  functional optimization must be proposed and reviewed separately after the
  corrected logs establish its cause.

## 4. Trace Contract

Use a new schema version so existing logs remain parseable:

```text
schema=requestbuffer-trace-v2
```

Every line must use whitespace-separated `key=value` fields. Print masks and
addresses in hexadecimal. Required common fields:

```text
event cycle tile bank window epoch entry
trace_id source_id address tag set opcode matrix_raw
valid ready fire reason
```

The unique lifecycle key is:

```text
tile + bank + window + epoch
```

`trace_id`, `source_id`, address, and entry are validation fields, not the sole
lifetime key. `epoch` is a monotonically increasing per-slice diagnostic ID
assigned when a request is allocated or directly flows through the buffer.

### Required events

| Event | Exact condition | Meaning |
| --- | --- | --- |
| `RB-INPUT` | Matrix `io.in.valid` | Input offer with `io.in.ready/fire` and all admission conditions |
| `RB-FLOW-FIRE` | Matrix `doFlow` | Request bypassed storage and fired directly to output |
| `RB-ALLOC` | Matrix `alloc && sel` | Entry allocation using incoming request and explicit next-state fields |
| `RB-ENTRY` | Valid matrix entry changes blocker/readiness state | Entry state transition, not an unconditional per-cycle dump |
| `RB-WAIT` | Valid matrix entry does not make progress | Per-cycle wait reason, including ready-but-not-selected cases |
| `RB-CHOSEN` | `chosenQ.io.enq.fire` | issueArb selection captured by chosenQ |
| `RB-CHOSEN-CANCEL` | `chosenQValid && cancel && chosenQ.io.deq.fire` | A real chosen entry was canceled; entry remains valid |
| `RB-OUTPUT` | Matrix `io.out.valid` | Output offer with ready/fire and selected origin (`flow` or `chosen`) |
| `RB-OUTPUT-FIRE` | Matrix `io.out.fire` | Definitive RequestBuffer-to-RequestArb admission |
| `RB-FREE` | Actual `e.valid := false` condition | Buffered entry was removed after non-canceled chosenQ fire |
| `RB-WINDOW-SUMMARY` | Trace window closes | Reconciled counts and blocker totals |

Do not emit `RB-FREE` for a canceled chosenQ event. A direct-flow request emits
`RB-FLOW-FIRE` and `RB-OUTPUT-FIRE`, but no `RB-ALLOC` or `RB-FREE`.

### Allocation fields

`RB-ALLOC` must never read newly assigned register values as if they were
already committed. Define explicit next-state expressions:

```text
alloc_rdy
alloc_wait_mp
alloc_wait_ms
alloc_mp_block_0
alloc_mp_block_1
alloc_s1a_block
alloc_s1b_block
alloc_mshr_conflict_mask
alloc_no_free_way
```

Print these expressions directly. If current entry register state is useful,
prefix it with `old_`; never mix it with `alloc_` fields.

### Per-entry state fields

`RB-ENTRY` and `RB-WAIT` must include:

```text
entry_valid entry_rdy wait_mp wait_ms
buffer_valid_mask buffer_rdy_mask
issue_eligible issue_selected
chosen_valid chosen_entry chosen_enq_fire chosen_deq_fire
real_cancel
mainpipe_block_0 mainpipe_block_1
s1a_block s1b_block
mshr_conflict_mask will_free_mask
no_free_way prefetch_blocked
out_valid out_ready out_fire
```

Define `real_cancel` as:

```text
chosenQValid && cancel
```

Never label raw `cancel=1` as a cancellation when `chosenQValid=0`.

### Reason encoding

The raw fields are authoritative. Use one stable summary code:

```text
0 NO_STALL
1 ALLOC_MAINPIPE_BLOCK
2 SAME_SET_S1A
3 SAME_SET_S1B
4 MSHR_CONFLICT
5 NO_FREE_WAY
6 PREFETCH_THROTTLE
7 ENTRY_NOT_READY_WAIT_MP
8 ENTRY_READY_NOT_SELECTED
9 CHOSENQ_FULL_OR_BUSY
10 CHOSEN_CANCELED
11 DOWNSTREAM_NOT_READY
12 BUFFER_FULL
13 UNKNOWN
```

If multiple blockers apply, preserve every raw field and select the first code
in the order above that explains lack of progress. Document the precedence in
both RTL comments and the parser.

## 5. RTL Tasks

- [ ] **R1. Add v2 common identity.** Expose tile, physical slice bank, trace
  window ID, and a per-slice diagnostic epoch to RequestBuffer logging without
  changing TaskBundle or TileLink payloads.
- [ ] **R2. Refactor blocker aliases.** Name the existing combinational
  expressions for `mpBlock`, `s1A_Block`, `s1B_Block`, conflict masks,
  `noFreeWay`, eligibility, selection, chosenQ movement, and real cancel. Reuse
  these aliases in logging; do not duplicate functional expressions with
  subtly different semantics.
- [ ] **R3. Correct allocation logging.** Print incoming request fields and
  explicit `alloc_*` next state. Remove ambiguous reads of `entry.rdy/waitMP`
  from the allocation event.
- [ ] **R4. Correct free/cancel events.** Emit `RB-FREE` only under the same
  condition that clears `e.valid`. Emit a separate `RB-CHOSEN-CANCEL` only when
  `chosenQValid && cancel` actually consumes/cancels chosenQ.
- [ ] **R5. Cover direct flow.** Emit a unique epoch and `RB-FLOW-FIRE` for
  every matrix request that bypasses buffer storage.
- [ ] **R6. Cover ready queueing.** Emit `RB-WAIT` when an entry is valid but
  cannot progress even if `e.rdy=1`, classifying issueArb/chosenQ/downstream
  causes.
- [ ] **R7. Add state transitions.** Emit `RB-ENTRY` when `rdy`, `waitMP`,
  `waitMS`, or relevant blocker state changes. This provides causality without
  printing unchanged entry state every cycle.
- [ ] **R8. Add a window summary.** Count input offers/fires, direct flows,
  allocations, true cancels, output fires, frees, wait request-cycles, and each
  reason code. Assert or report:

  ```text
  output_fires = direct_flows + frees
  allocations = frees + entries_still_valid_at_window_end
  ```

## 6. Parser Tasks

- [ ] **P1. Parse v1 and v2 separately.** Preserve best-effort v1 support but
  mark its lifecycle analysis incomplete because it lacks epochs and definitive
  events.
- [ ] **P2. Reconstruct by epoch.** Group v2 records by
  `(tile, bank, window, epoch)`, validate constant trace/source/address/entry,
  and reject mixed lifetimes.
- [ ] **P3. Support both lifecycle shapes.** Accept exactly:

  ```text
  RB-INPUT -> RB-FLOW-FIRE -> RB-OUTPUT-FIRE
  RB-INPUT -> RB-ALLOC -> [RB-ENTRY/RB-WAIT/RB-CHOSEN/CANCEL]*
           -> RB-OUTPUT-FIRE -> RB-FREE
  ```

- [ ] **P4. Reconcile events.** Report duplicates, free-without-alloc,
  alloc-without-free, output-fire-without-input, canceled-and-freed-same-cycle,
  changed identity fields, and window summary mismatches.
- [ ] **P5. Classify waits.** Produce first blocker, dominant blocker, blocker
  request-cycle counts, ready-but-not-selected cycles, true cancels, and total
  input-to-output latency for every request.
- [ ] **P6. Add batch/bank correlation.** Join v2 epochs to MEM trace IDs and
  `L2-MATRIX-OUT-CYCLE` masks. For the first partial mask, list delayed banks,
  epochs, blocker timelines, and preceding same-set requests.
- [ ] **P7. Separate evidence from inference.** JSON/Markdown output must have
  `direct_evidence`, `classification`, `excluded_causes`, `confidence`, and
  `evidence_lines` fields.
- [ ] **P8. Avoid broken-pipe failures.** CLI output should terminate cleanly
  when piped through tools such as `head`.

Recommended outputs:

```text
requestbuffer_trace_report.json
requestbuffer_trace_report.md
requestbuffer_lifetimes.csv
requestbuffer_wait_intervals.csv
requestbuffer_cycle_masks.csv
```

## 7. Tests

### RTL tests

- [ ] Direct-flow matrix request emits input, flow, and output-fire with one
  epoch and never emits alloc/free.
- [ ] Buffered request emits exactly one alloc and one true free.
- [ ] A real chosenQ cancel emits cancel but not free; reselection later closes
  the same epoch.
- [ ] `chosenQValid=0 && cancel=1` emits no cancel event.
- [ ] Same-set S1A and S1B cases set distinct raw fields and reason codes.
- [ ] MainPipe block, MSHR conflict, no-free-way, chosenQ busy, and downstream
  ready-low cases are independently distinguishable.
- [ ] Two requests reusing the same trace ID and entry receive different epochs.
- [ ] Trace disabled/enabled comparison has identical ready/valid, issue choice,
  buffer state, and output timing.

### Parser tests

- [ ] Parse one direct-flow and one buffered lifecycle.
- [ ] Handle cancel/reselection without duplicate free.
- [ ] Keep reused trace/entry IDs separate by epoch.
- [ ] Detect every malformed lifecycle listed in P4.
- [ ] Aggregate simultaneous events from all eight banks without loss.
- [ ] Reproduce the 0903 first-deficit explanation from a compact fixture.

### Workload acceptance

- [ ] Run the same matrix-no-probe workload and retain 10,240 complete MEM
  traces and functional success.
- [ ] Reconcile every matrix `RB-INPUT.fire` with exactly one
  `RB-OUTPUT-FIRE`.
- [ ] Reconcile all `RB-ALLOC` with one non-canceled `RB-FREE`; no parser-open
  lifetimes remain when the trace window closes normally.
- [ ] For the 527515-style bank 3/7 event, report the actual allocation
  blocker, waitMP transitions, true selection/cancel history, and output fire.
- [ ] Classify the remaining 4,269 allocations instead of leaving them as
  ambiguous `reason=5` events.
- [ ] Confirm cache miss, replacement, and non-matrix traffic remain zero for
  the controlled run.

Required validation commands:

```bash
python -m unittest scripts.tests.test_requestbuffer_service_trace
make comp -j4
mill -i xiangshan.checkFormat
make sim-verilog CONFIG=DefaultMatrixConfig
```

Run focused RequestBuffer tests before the full simulation build. Disable
ccache for Verilator tests when the environment's global ccache directory is
read-only.

## 8. Documentation Deliverable

Update `docs/requestbuffer-service-log-analysis-guide.md` after implementation
to contain the final v2 schema, event examples, reason precedence, lifecycle
validation rules, and an end-to-end worked example from the new run. Remove or
clearly label all v1 assumptions, especially:

- treating raw `cancel` as a real cancel;
- treating `RB-FREE` as reliable when `cancel=1`;
- grouping solely by trace ID and entry;
- treating every non-`0xff` downstream mask as response loss;
- treating residual `has_clients` metadata as a new L1 request.

The updated guide must enable another agent to analyze a debug log without
access to the RTL source.
