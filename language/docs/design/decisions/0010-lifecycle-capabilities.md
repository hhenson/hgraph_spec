# ADR 0010: clock and scheduler capabilities, scheduled handlers, and input activity

Status: accepted.

## Context

The language agreed the injectable names `clock` and `scheduler` with no
surface behind them, and the migration catalogue holds thirty-six core
operators under blocker B2 (lifecycle and activation). Three things were
missing, and they are what the native stream nodes actually use:

- an evaluation clock (`EvaluationClockView`: evaluation time, wall-clock
  time, next cycle);
- the node scheduler (`NodeScheduler`: schedule after a delay or at a time,
  on the engine or the wall clock; query the next alarm; know whether the
  current evaluation is the alarm firing); and
- input activity: a node that has seen enough ticks stops listening to an
  input (`make_passive`) and may listen again later (`make_active`).

The language model also left two lifecycle questions open: the meaning of a
bare handler in a runtime function with no temporal parameters, and a source
form for an explicitly empty activation set (requirement MIG-014, library
blocker LIB-002). Both are the scheduler case.

## Decision

Clock observations use [read-only properties](../clock-properties.md).
Capability actions and the scheduler queries below use
[receiver-first functions](../capability-function-syntax.md); no receiver
method-call aliases are admitted.

1. **`inject clock`** provides the evaluation clock as `clock` in every
   hook. `clock.evaluation_time`, `clock.now` and
   `clock.next_cycle_evaluation_time` are read-only `datetime` properties.
   Evaluation time is the current cycle's logical time and remains constant
   throughout that cycle. Start uses the run's start time; stop retains the
   last cycle's time, or the start time if no cycle ran.
   Next-cycle evaluation time is evaluation time plus
   the engine's smallest step and is equally stable within the cycle. Now is
   the engine's current wall-time estimate: UTC computer time in real time.
   In simulation, the run's first `clock.now` read in any lifecycle phase
   establishes an origin and returns evaluation time. Later reads add real
   elapsed time from that origin; subsequent cycle starts replace it, while
   phase boundaries retain it (see [clock timing](../clock-properties.md#timing-semantics)).
   Neither method-call nor free-function aliases are admitted.

2. **`inject scheduler`** provides the node scheduler as `scheduler` in
   every hook. `schedule(scheduler, delay)` and
   `schedule(scheduler, delay, on_wall_clock)` take a `duration`;
   `schedule_at(scheduler, time)` and `schedule_at(scheduler, time, on_wall_clock)`
   take a `datetime`; `is_scheduled(scheduler)` returns `bool`;
   `next_scheduled_time(scheduler)` returns `datetime`. Wall-clock alarms
   follow hgraph's rule: only a real-time executor accepts them, and a due
   alarm fires on the next evaluatable cycle. Tags are not exposed.

3. **`scheduled()` is a handler selector.** It is valid only in a
   function-level `when` condition of a function that injects `scheduler`,
   and is true when the current evaluation is the node's alarm firing
   (`is_scheduled_now`). A source on the stateless `alarm` of
   [ADR 0015](0015-pull-sources.md) does not use it: it publishes from a
   plain `when`, since every evaluation is its wake-up. A handler with `scheduled()` at top level receives
   no implicit `modified()`; its implicit `valid()` is unchanged. Such a
   handler adds no input to the node's activation set. When no handler names
   an input, the set is explicitly empty: every input is passive and the
   node evaluates only when scheduled. This is the agreed spelling of the
   empty activation set; `modified()` keeps its complete-list meaning.

4. **A runtime function may have no temporal parameters when it injects
   `scheduler`.** It is a source: `start { schedule(scheduler, 0s) }` asks
   for evaluation in the starting cycle, which is how a static node's
   `schedule_on_start` is spelled. The implicit `valid()` of such a
   function is vacuously true and its implicit `modified()` never holds.
   [ADR 0015](0015-pull-sources.md) extends this decision: a source may
   instead inject the stateless `alarm`, or be a generator written with
   `yield`.

5. **`passivate(input)` and `activate(input)`** are runtime statements
   whose argument is a direct temporal parameter of the function. They are
   evaluation-only: `start` and `stop` have no temporal input access. They
   lower to the input view's `make_passive()` / `make_active()`. A passive
   input still holds its value and validity; it just no longer activates the
   node. The node's static activation policy is unchanged; activity is a
   per-evaluation effect, exactly as in a hand-written node.

## Consequences

- `freeze`, `take` and `until_true` become authorable as parallel identities.
  `schedule` remains blocked: its native counter is non-recordable `State<Int>`,
  while HGL `state` is checkpointed. The cache declarations in ADR 0008 and
  positive-delay validation in `start` are still required.
  Likewise, `throttle`, `batch`, `gate`, `lag` and `window` still
  need the buffered-delta and queue state contracts (MIG-005) and remain B2.
- The language model's open questions on bare handlers without temporal
  parameters and on the explicit empty activation set are closed by
  decisions 3 and 4. LIB-002 (collection startup) can now be spelled with
  `start { schedule(scheduler, 0s) }` and `when scheduled()`; closing it is
  library work, not language work.
- `scheduled`, `passivate` and `activate` are intrinsic names and cannot be
  used as identifiers.
- Output access inside `start` and `stop` remains rejected; scheduling from
  `start` is admitted because the scheduler is a capability, not the output.
