# Engine cases

Status: proposed 2026-09-30, reasoned from ENG-1 to ENG-16; not yet run.
Times are offsets from the run's start time in smallest steps. `—` means no
cycle.

## ENGINE-BOUNDS — ENG-1, ENG-3, ENG-5

Simulation. A source on the scheduler publishes at 0, 3, 6 and 9 (it
requests the start time in start, then 3 cycles on each time). The run's end
time is 6.

| Offset | Cycle runs | Published |
|---|---|---|
| 0 | yes | 0 |
| 3 | yes | 3 |
| 6 | no: the end time is exclusive | — |

The run ends when the next scheduled time is not before the end time. The
source is stopped once; its pending request is discarded with it. A second
run from the same description starts fresh at 0.

## ENGINE-NOTHING-SCHEDULED — ENG-5

Simulation. A source publishes at 0 and asks for nothing more. The run ends
after cycle 0 although the end time is far later: a simulation with nothing
scheduled is finished. In real time the same graph would wait for an outside
event or the end time.

## ENGINE-NO-SKIP — ENG-4

Simulation. Two sources: one due at 1, 2, 3, 4; another at 2 and 4. Four
cycles run, at 1, 2, 3 and 4; at 2 and 4 both sources are evaluated in one
cycle. No time is skipped and no two are merged.

## ENGINE-STOP-REQUEST — ENG-9, ENG-10, INJ-10

A source publishes every cycle from 0. A sink requests a stop when it sees
the value 2 (at offset 2). The cycle at 2 completes: every node scheduled for
2 after the sink in rank order is still evaluated. No cycle runs at 3. Every
started node is stopped, in reverse rank order.

## ENGINE-FAILURE — ENG-10, NOD-20, Engine point 1

A node's eval fails at 1 without capture. The failure leaves the root graph;
no further cycle runs; every started node is stopped in reverse rank order;
the report names the node and the phase (evaluate). In a second variant the
failing node's start fails: the nodes started before it are stopped in
reverse order, it is not stopped, and the report names the phase (start).

## ENGINE-CLOCK — ENG-2, ENG-11, ENG-13, ENG-14, INJ-7

Simulation, cycles at 0 and 5. In each cycle two nodes of different rank
read the clock.

| Reading | Both nodes see |
|---|---|
| evaluation time | the cycle's time: 0, then 5 |
| next cycle evaluation time | one smallest step on: 1, then 6 |
| now | not before the evaluation time; the second node's reading is not before the first's |
| lag | not negative; the second node's reading is not less than the first's |

Nothing a node does writes the clock (ENG-11). The first *now* or *lag*
read activates simulation sampling as specified by ENG-14; see
[ENGINE-CLOCK-LAZY](#engine-clock-lazy--eng-13-eng-14).

## ENGINE-REAL-TIME-ORDER — ENG-2, ENG-6, ENG-8

Real time. A scheduled source is due at wall-clock time T; a push source's
sender admits an event just before T. The event's cycle runs first, at a
time not later than T; the scheduled cycle runs at T or one smallest step
after the event's cycle, whichever is later. Evaluation time never repeats
and never decreases. When the engine is late for T it evaluates at once.

## ENGINE-NESTED-CLOCK — ENG-12

A nested graph's node reads the same evaluation time and next cycle time as
a root node in the same cycle.

## Outside these cases

Externally driven stepping, observers, one-shot callbacks, the runaway guard
and pausing are optional facilities with no cases
here. The order of stop and report on failure (Engine point 1) is not
observable by a graph and has no case.

## ENGINE-CLOCK-LAZY — ENG-13, ENG-14

Use a controlled real-time timer for the engine clock that records sample
requests and advances only when the test moves it. This is a test fixture, not a new graph-facing
clock API. Timer values below are seconds from a fixed UTC origin W, independent of
logical evaluation time E. Hold the timer fixed while taking paired property reads.

Run these simulation steps twice, once with *lag* as the first property read
and once with *now* first:

| Step | Required observation |
|---|---|
| First cycle: read only evaluation time and next cycle time | No timer samples |
| Second cycle begins at timer 10; advance to 17 without reading *now* or *lag* | Still no timer samples |
| At 17, first read followed by the other property | Lag is 0; now is E; sampling has begun |
| Advance to 20 in the same cycle; read both | Lag is 3s; now is E + 3s |
| Next cycle begins at 30; first property reads at 37 | A sample at cycle start; lag is 7s, now is E + 7s |
| Next cycle begins at 40; neither property is read | A sample at cycle start |
| Next cycle begins at 50; first property reads at 55 | Lag is 5s; now is E + 5s |

A separate simulation with no *now* or *lag* reads takes no engine-clock
elapsed-time samples throughout start, all cycles and stop. Sample counts
cover the run only, not clock construction before it. In real-time mode, a cycle beginning at timer 10 and
first observed at 17 instead has lag 7s; *now* is W + 17s.
Real-time scheduling may sample before any property read.

An eager simulation sampler fails both the no-sample checks and the first
lag-zero check: its origin at 10 would produce lag 7 at 17. Resetting the
origin on every cycle's first property read fails the later lag-7 check.
These exact values rely on the controlled timer, not on physical-clock
sampling overhead.

## ENGINE-CLOCK-LIFECYCLE — ENG-13, ENG-14, INJ-7

Use the same controlled timer. Each row is a separate run; S is the run's
logical start time, and E is its most recent cycle's evaluation time. For
each first-read scenario, run both *now*-first and *lag*-first variants,
holding the timer fixed until both properties have been read.

| Scenario | Required observations |
|---|---|
| Startup begins at 10; first read at 17, another at 20; first cycle begins at 30, read at 37 | No elapsed-time samples before 17. At 17 lag is 0 and now is S; at 20 lag is 3s and now is S + 3s. The cycle samples at 30; at 37 lag is 7s and now is E + 7s |
| Cycles run without either property read; first read is in stop at 50, another at 53 | No elapsed-time samples before 50. At 50 lag is 0 and now is E; at 53 lag is 3s and now is E + 3s |
| No cycle runs; first read is in stop at 50, another at 53 | No elapsed-time samples before 50. At 50 lag is 0 and now is S; at 53 lag is 3s and now is S + 3s |
| Sampling was activated in startup; last cycle begins at 30; stop reads at 40 | Lag is 10s and now is E + 10s: stop did not reset the cycle origin |
| First read in startup at 17; no cycle runs; stop reads at 40 | Lag is 23s and now is S + 23s: stop retained the startup read's origin |

Real-time controls begin root-graph startup at 10. A startup read at 17 has
lag 7s and now W + 17s. If no cycle runs, a stop read at 40 has lag 30s;
if the last cycle began at 30, it instead has lag 10s. Both stop variants
read now W + 40s, retaining evaluation time S or E respectively.
