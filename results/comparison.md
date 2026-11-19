# LOC vs LOS results

Measured on `cadence-server.vit.ac.in` with **Modus 22.10-p008_1 (built Oct 18 2022)**, design `top` from
Project 2 (`dft-scan-atpg`: a synchronous FIFO `fifo_sync_W8_AW2`, 8-bit data, 4-entry, inside a `top`
wrapper, with 1 scan chain of 129 bits), run directory `runs/modus-20261119-001804`. Both transition runs use the same netlist, library, fault model and
scan structure; they differ only in the dynamic sequence filter and the scan-enable release.

Coverage is the dynamic-fault figure. Modus reports `%TCov = #Tested/#Faults` and
`%ATCov = #Tested/(#Faults-#Redund)`; with 0 redundant faults these are numerically identical, and the
values below are quoted from `atpg.stats:dynamic.atcov` (global) and the experiment-level `tcov`
(testmode scope) in `tbdata/modus_metrics`. The static rows the same shared fault model also prints
are not used.

| Metric | Project 2 stuck-at | Transition LOC (`intra`) | Transition LOS (`losintra`) |
|---|---:|---:|---:|
| Total faults (testmode scope) | 4556 | 4164 | 4164 |
| Detected faults | 4556 | 2805 | 3377 |
| Test coverage % (testmode scope) | 100.00 | **67.36** | **81.10** |
| Pattern count (scan + logic) | 91 (91 + 0) | 106 (1 + 105) | 64 (1 + 63) |
| ATPG runtime (s) | 1.47 | 7.02 | 4.79 |
| Untestable / ATPG-untestable (redundant + possible) | 0 | 0 | 0 |
| Undetected (aborted/untested) | 0 | 1359 | 787 |
| Global-scope dynamic faults | — | 4556 | 4556 |
| Global test coverage % (all dynamic faults) | — | 61.57 | 74.12 |
| Modus release | 22.10-p008_1 | 22.10-p008_1 | 22.10-p008_1 |
| Run directory | `p2_atpg/runs/c1` | `runs/modus-20261119-001804` (`loc_exp`) | `runs/modus-20261119-001804` (`los_exp`) |

**Why there are two denominators.** `build_faultmodel -includedynamic yes` creates 4556 dynamic faults on
this design. The default testmode-scope dynamic exclusions (`-excludedynamicscanenable testmode`,
`-excludedynamicsetreset testmode`) inactivate 392 of them because `SE` is a test-function pin, leaving
4164 active faults in `TDF_LOC` and `TDF_LOS`. Both modes have the *same* 4164-fault scope, so the
67.36% / 81.10% testmode numbers are directly comparable; the global 61.57% / 74.12% numbers use all 4556.
The Project 2 stuck-at row is 4556 faults, so compare it with the global dynamic coverage, not the
testmode coverage.

**Delta (testmode scope):** LOS detects 572 more faults (+13.74 percentage points) with 42 fewer patterns
(−39.6%) and a shorter ATPG run (4.79 s vs 7.02 s). On the global denominator LOS is +12.55 points.

**Flow caveat.** No clock-constraints file was supplied, so this is Modus's untimed dynamic flow
(`-dynseqfilter` selects the clocking). The launch/capture sequences and scan-enable behaviour are
generated correctly, but no at-speed period is modelled. A timed flow needs `read_sdc` or a
`-clockconstraints` file; the coverage/pattern comparison is unaffected, absolute timing signoff is not.

Raw evidence: `results/evidence/` (`kpi.txt`, `modus_metrics.txt`, both `verify_test_structures` logs,
both `create_logic_delay_tests` logs, and the full console log). Vectors are in
`runs/modus-20261119-001804/vectors/`.

## Analysis

### Why is transition coverage lower than stuck-at coverage?

Stuck-at ATPG on this design reaches 100% of 4556 faults; LOC reaches 67.36% of the 4164 dynamic faults
in scope. A transition fault needs a two-time-frame test: the launch value must be justified, an event
must propagate through combinational logic, and the result must be captured one functional period later
at a register or output. Stuck-at requires only a single static assignment. The gap here is dominated by
the design's structure: it is a small synchronous FIFO, so its 129 scan bits are mostly the data array
plus the read/write pointers and full/empty flags, and those flops are wired into wide, largely parallel
compare and select logic. Many transition faults in that logic need a launch value that the pointer and
flag logic cannot present in a single functional cycle, so LOC ends up aborting them rather than proving
them undetectable. Modus classifies none of the misses as untestable —
`possible_faults` and `redundant_faults` are 0 — so the 1359 undetected LOC faults were *aborted* under
the default high effort, not proven untestable, which is exactly the pool LOS later recovers.

### Why does LOS cover more faults with fewer patterns?

LOS launches from the last shift instead of from a functional capture cycle, so the launch value does not
have to be justified through the functional logic: any pair the scan chain can shift in is available.
That removes the functional-reachability constraint that limits LOC and lets the delay generator abort
fewer faults. The measured effect is large here — 3377 vs 2805 detected (+572), and 787 vs 1359 undetected.
The same freedom makes each pattern more efficient, so LOS needs 64 tests where LOC needs 106. LOS is
harder to deploy because `SE` must leave the scan state between the launch and capture edges at
functional speed, which demands a balanced, timing-closed scan-enable tree; the comparison here uses the
same Project 2 netlist for both modes and therefore does not verify that path.

### What is over-testing, and which method risks it more?

Over-testing is rejecting a good device for a transition that functional operation cannot produce. LOS
risks it more: a two-bit pattern that is simply shifted in can pair states that the FSM or datapath never
reaches, so it may measure delay along non-functional paths. LOC launches through a functional capture
cycle, so its launch states are at least functionally reachable, at the cost of the coverage shown above.
For this design the 572 extra LOS detections should be reviewed against functional reachability — in
particular through the FIFO's pointer and full/empty decode logic, where shifted-in launch pairs can
produce states the FIFO never enters — before treating the higher number as strictly better.

### Would LOC or LOS be the production choice for this design?

Neither is justified by coverage alone. LOC is the lower-risk default: `SE` can settle before the
at-speed capture sequence, so it needs no additional scan-enable timing closure, and it still covers
67.36% of the in-scope dynamic faults with 106 patterns. LOS is worth its extra hardware and timing
burden only if the 572 additional detected faults are functionally meaningful and the scan-enable path is
implemented and verified for at-speed switching — which this project did not do, since both modes run on
the same Project 2 netlist. A production decision also needs the timed flow (SDC constraints or a Modus
clock-constraints file), the false/multi-cycle path exclusions, and a check of the LOS launch states
against the design's legal operation.

## Evidence to attach

- Modus release and complete console log: `results/evidence/modus_console.log`.
- `verify_test_structures` results for both test modes: `results/evidence/verify_TDF_LOC.log`,
  `results/evidence/verify_TDF_LOS.log` (chain 129 bits, controllable/observable, no errors).
- Dynamic fault totals and status per mode: `results/evidence/kpi.txt`, `results/evidence/modus_metrics.txt`.
- `create_logic_delay_tests` logs: `results/evidence/delay_tg_TDF_LOC.log`, `.../delay_tg_TDF_LOS.log`.
- Project 2 stuck-at KPI: `results/evidence/p2_stuckat_kpi.txt`.
- The exact pin assignment files used: `inputs/top_loc.pinassign`, `inputs/top_los.pinassign`.
- Vectors: `runs/modus-20261119-001804/vectors/{loc,los}_transition.*` (kept out of git by `.gitignore`).
