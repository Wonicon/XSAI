# Tasks: Cache Matrix Activity Tracing

**Input**: Design documents from `specs/007-cache-matrix-tracing/`

**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`,
`contracts/cache-matrix-trace-v1.md`, and `quickstart.md`.

## Phase 1: Setup and Context

- [X] T001 Record current XSTile, CUTE control, CoupledL2, Slice, SinkA, Directory, MainPipe, MSHR, and test revisions in `specs/007-cache-matrix-tracing/quickstart.md`
- [X] T002 [P] Confirm `DefaultMatrixConfig`, matrix-enabled XSTile, L2 bank count, tag/set/bank/offset widths, and debug switches in `src/main/scala/top/Configs.scala` and `XSAICache/src/main/scala/coupledL2/L2Param.scala`
- [X] T003 [P] Trace actual `AmuCtrl.fire` and core-side `mrelease.fire` ownership in `src/main/scala/xiangshan/XSCore.scala`, `src/main/scala/xiangshan/XSTile.scala`, and `CUTE/src/main/scala/TaskController.scala`
- [X] T004 [P] Trace MatrixKey, AmeIndexKey, ReqSourceKey, `matrixTask`, `ReplacerInfo`, and MSHR metadata propagation in `CUTE/src/main/scala/CUTE2YGJK.scala` and `XSAICache/src/main/scala/coupledL2/{SinkA.scala,Common.scala,MSHR.scala}`
- [X] T005 [P] Confirm scalafmt/scalastyle scope and XSAICache/CUTE Mill test targets in `AGENTS.md`, `XSAICache/build.sc`, and `CUTE/build.sc`

## Phase 2: Foundational Contracts and Fixtures

- [X] T006 [P] Freeze event names, common fields, sequence rules, and summary contract in `specs/007-cache-matrix-tracing/contracts/cache-matrix-trace-v1.md`
- [X] T007 [P] Add configured-width tag/set/bank/offset address reconstruction oracle in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceTestUtils.scala`
- [X] T008 [P] Add MatrixKey origin decoder fixture for `00/01/10/11` and unknown handling in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceTestUtils.scala`
- [X] T009 [P] Add external/internal request identity fixture covering source class, channel, opcode, retry, and refill in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceTestUtils.scala`
- [X] T010 Define trace build/debug option defaults and configuration invariants in `XSAICache/src/main/scala/coupledL2/L2Param.scala`
- [X] T011 Add parser fixtures for missing, duplicate, out-of-order, unclosed, and summary-mismatch records in `scripts/tests/test_cache_matrix_trace.py`
- [X] T012 Validate foundational fixtures against `specs/007-cache-matrix-tracing/data-model.md` and `specs/007-cache-matrix-tracing/contracts/cache-matrix-trace-v1.md`

## Phase 3: User Story 1 - Bound the Trace Window (Priority: P1)

**Independent Test**: XSTile chiseltest proves one start on first accepted
`AmuCtrl`, persistence through release stall, one end on accepted release,
reset abort, and tile isolation.

- [X] T013 [P] [US1] Add reset, first-control, repeated-control, and start-record test in `src/test/scala/xiangshan/XSTileCacheTraceSpec.scala`
- [X] T014 [P] [US1] Add release-stall, release-fire, reset-abort, and post-window exclusion test in `src/test/scala/xiangshan/XSTileCacheTraceSpec.scala`
- [X] T015 [P] [US1] Add per-tile isolation test in `src/test/scala/xiangshan/XSTileCacheTraceSpec.scala`
- [X] T016 [US1] Add per-tile active state, window ID, sequence, counters, and reset behavior at the control boundary in `src/main/scala/xiangshan/XSTile.scala`
- [X] T017 [US1] Qualify activation with first `AmuCtrl.fire` and preserve it across later controls and release backpressure in `src/main/scala/xiangshan/XSTile.scala`
- [X] T018 [US1] Qualify deactivation with core-side `mrelease.fire` and define release-cycle inclusion in `src/main/scala/xiangshan/XSTile.scala`
- [X] T019 [US1] Propagate active/window identity and lifecycle boundaries through existing wiring in `src/main/scala/xiangshan/XSTile.scala` and `XSAICache/src/main/scala/coupledL2/CoupledL2.scala`
- [X] T020 [US1] Emit `CACHE-WINDOW-START` and `CACHE-WINDOW-END` records and summary defaults without affecting cache control in `src/main/scala/xiangshan/XSTile.scala` and `XSAICache/src/main/scala/coupledL2/CoupledL2.scala`
- [X] T021 [US1] Run XSTile window tests and record cycle-boundary evidence in `specs/007-cache-matrix-tracing/quickstart.md`

## Phase 4: User Story 2 - Attribute Replacements (Priority: P1)

**Independent Test**: Directory/MSHR fixtures reconcile every valid clean/dirty
victim exactly once, exclude invalid-way allocation, and preserve original
trigger metadata across retry/refill.

- [X] T022 [P] [US2] Add invalid-way versus valid clean/dirty victim fixture in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceSpec.scala`
- [X] T023 [P] [US2] Add simultaneous-bank replacement and address reconstruction checks in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceSpec.scala`
- [X] T024 [P] [US2] Add retry/refill causal-trigger identity fixture in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceSpec.scala`
- [X] T025 [US2] Capture accepted external trigger metadata in `XSAICache/src/main/scala/coupledL2/SinkA.scala` and `RequestArb.scala`
- [X] T026 [US2] Observe committed victim validity/state/dirty, selected way, and trigger metadata in `XSAICache/src/main/scala/coupledL2/Directory.scala` and `MainPipe.scala`
- [X] T027 [US2] Carry trigger tag/set/bank, matrix metadata, source, channel, opcode, and external identity through `XSAICache/src/main/scala/coupledL2/Common.scala` and `MSHR.scala`
- [X] T028 [US2] Implement configured-width victim/trigger address formatting in `XSAICache/src/main/scala/coupledL2/CacheMatrixTrace.scala`
- [X] T029 [US2] Emit `CACHE-REPLACE` only for valid victims and update clean/dirty summary counters in `XSAICache/src/main/scala/coupledL2/Directory.scala`, `MainPipe.scala`, and `CacheMatrixTrace.scala`
- [X] T030 [US2] Run replacement fixtures and record exact event/source/address reconciliation in `specs/007-cache-matrix-tracing/quickstart.md`

## Phase 5: User Story 3 - Identify Matrix Misses (Priority: P1)

**Independent Test**: Matrix hits, normal CUTE misses, accumulator misses,
L1/non-CUTE misses, and internal replay produce exactly one event per external
miss identity.

- [X] T031 [P] [US3] Add MatrixKey hit/miss tests for `01`, `11`, `00`, and reserved `10` in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceSpec.scala`
- [X] T032 [P] [US3] Add replay/refill non-duplication test in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceSpec.scala`
- [X] T033 [P] [US3] Add miss address/source/opcode/way reconciliation checks in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceSpec.scala`
- [X] T034 [US3] Preserve raw MatrixKey and decode CUTE/L1/unknown origin beside `matrixTask` in `XSAICache/src/main/scala/coupledL2/SinkA.scala`
- [X] T035 [US3] Define first-observation external miss identity at directory result handling in `XSAICache/src/main/scala/coupledL2/MainPipe.scala`
- [X] T036 [US3] Carry miss identity through MSHR allocation/retry without changing arbitration in `XSAICache/src/main/scala/coupledL2/MSHR.scala` and `MSHRCtl.scala`
- [X] T037 [US3] Emit `CACHE-MATRIX-MISS` with address, origin, source, opcode, way, and `needs_victim` in `XSAICache/src/main/scala/coupledL2/MainPipe.scala` and `CacheMatrixTrace.scala`
- [X] T038 [US3] Update miss summary counters and suppress hit/replay duplicates in `XSAICache/src/main/scala/coupledL2/CacheMatrixTrace.scala`
- [X] T039 [US3] Run matrix-miss fixtures and record exact-once evidence in `specs/007-cache-matrix-tracing/quickstart.md`

## Phase 6: User Story 4 - Expose Non-Matrix Interference (Priority: P2)

**Independent Test**: Ingress tests cover demand, prefetch, PTW, coherence,
maintenance, retry/refill, accepted requests, and multi-cycle stalls; summaries
match independent cycle observations.

- [X] T040 [P] [US4] Add source-class tests for all configured `MemReqSource` values in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceSpec.scala`
- [X] T041 [P] [US4] Add valid-held/ready-low stall and fire-correlation tests in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceSpec.scala`
- [X] T042 [P] [US4] Add outside-window exclusion and end-summary reconciliation tests in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceSpec.scala`
- [X] T043 [US4] Observe non-matrix ingress valid/ready/fire, address, source, channel, opcode, and raw matrix metadata in `XSAICache/src/main/scala/coupledL2/SinkA.scala` and `RequestArb.scala`
- [X] T044 [US4] Decode CPU/L1 demand, prefetch, PTW, coherence, maintenance, retry/refill, and unknown classes in `XSAICache/src/main/scala/coupledL2/CacheMatrixTrace.scala`
- [X] T045 [US4] Emit `CACHE-NONMATRIX-REQ` per active-window valid cycle without feeding observation state into ready/arbitration in `XSAICache/src/main/scala/coupledL2/CacheMatrixTrace.scala`
- [X] T046 [US4] Update offered, accepted, stalled-cycle, and source-class summary counters in `XSAICache/src/main/scala/coupledL2/CacheMatrixTrace.scala`
- [X] T047 [US4] Run interference fixtures and record raw/summary count reconciliation in `specs/007-cache-matrix-tracing/quickstart.md`

## Phase 7: Parsing, Integration, and Signoff

- [X] T048 [P] Implement lifecycle, address, exact-once, and incomplete-evidence parser in `scripts/cache_matrix_trace_report.py`
- [X] T049 [P] Add parser tests for valid and malformed `cache-matrix-trace-v1` fixtures in `scripts/tests/test_cache_matrix_trace.py`
- [X] T050 Integrate active/window identity across all CoupledL2 slices/banks without changing TileLink payloads in `XSAICache/src/main/scala/coupledL2/CoupledL2.scala` and `Slice.scala`
- [X] T051 Compare trace-enabled/disabled handshakes, replacement choices, coherence outcomes, responses, and timing in `XSAICache/src/test/scala/coupledL2/CacheMatrixTraceSpec.scala`
- [X] T052 Run focused cache/XSTile tests, `make check-format`, and targeted compile; record results in `specs/007-cache-matrix-tracing/quickstart.md`
- [X] T053 Run `make comp` and `make sim-verilog CONFIG=DefaultMatrixConfig`; inspect generated RTL for latches, loops, width issues, and disabled-trace behavior in `specs/007-cache-matrix-tracing/quickstart.md`
- [X] T054 Run deterministic real-L2 zames workload with `WITH_CHISELDB=1`, capture the control-to-release window, and record logs/waveform paths in `specs/007-cache-matrix-tracing/quickstart.md`
- [X] T055 Correlate long-latency CUTE requests with matrix misses, replacements, non-matrix interference, L2 service latency, and request-fire bandwidth in `scripts/cache_matrix_trace_report.py`
- [X] T056 Verify real-L2 functional `GOOD TRAP`, closed trace lifecycle, summary reconciliation, and incomplete diagnostics in `specs/007-cache-matrix-tracing/quickstart.md`
- [X] T057 Update spec, plan, research, data model, contract, and quickstart with final field names and validation evidence in `specs/007-cache-matrix-tracing/`

## Phase 8: Real-L2 Service Perturbation Diagnostics

- [X] T058 Add trace-window RequestArb, directory/MainPipe, GrantBuffer, and CustomL1Hint service events in XSAICache
- [X] T059 Add per-cycle L2 matrix output and CUTE ResponseBridge/Omega/loader-facing masks
- [X] T060 Add `scripts/cache_service_trace_report.py` and focused first-deficit parser tests
- [X] T061 Document the event dictionary, causal analysis procedure, and 527522 example in `docs/real-l2-service-perturbation-log-analysis.md`
- [X] T062 Run focused parser tests, Scala compile, and `mill -i xiangshan.checkFormat`; RTL elaboration completed with existing width warnings, then Difftest file collection stopped because `NOOP_HOME` was unset

## Dependencies and Execution Order

- Setup T001-T005 has no prerequisite.
- Foundational T006-T012 depends on Setup and blocks RTL work.
- US1 T013-T021 depends on foundational lifecycle/identity contracts.
- US2 T022-T030 depends on US1 propagation and can run in parallel with US3/US4 fixtures.
- US3 T031-T039 depends on US1 and shared origin/address contracts.
- US4 T040-T047 depends on US1 and source classification, independent of replacement RTL.
- Signoff T048-T057 depends on all selected stories and the parser contract.

## Parallel Execution Examples

```text
After T012: T013-T015 || T022-T024 || T031-T033 || T040-T042
After T020: T025-T029 || T034-T038 || T043-T046
After all stories: T048-T049 || T051 || T052
```

## Implementation Strategy

MVP is US1 plus the foundational contracts. The first diagnostic release should
then complete US2 and US3, because replacement and matrix-miss attribution are
needed to explain real-L2 latency. US4 adds interference attribution, followed
by parser, disabled-trace equivalence, RTL generation, and real-L2 signoff.
