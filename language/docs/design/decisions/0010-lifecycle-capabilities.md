# ADR 0010: clock and scheduler capabilities, scheduled handlers, and input activity

Status: accepted.

## Context

The `clock` and `scheduler` capabilities and input activity cover:

- evaluation time, wall-clock time and next-cycle time;
- scheduling after a delay or at a time, on the engine or wall clock,
  querying the next alarm and identifying an alarm-driven evaluation; and
- stopping and resuming notifications from an input with `passivate` and
  `activate`.

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
   In simulation, the shared clock's first `now` or runtime-only `lag` read
   in any lifecycle phase establishes the origin. `clock.now` returns evaluation
   time if it establishes that origin; otherwise it adds elapsed time from it.
   Subsequent cycle starts replace the origin, while
   phase boundaries retain it (see [clock timing](../clock-properties.md#timing-semantics)).
   Neither method-call nor free-function aliases are admitted.

2. **`inject scheduler`** provides the node scheduler as `scheduler` in
   every hook. `schedule(scheduler, delay)` and
   `schedule(scheduler, delay, on_wall_clock)` take a `duration`;
   `schedule_at(scheduler, time)` and `schedule_at(scheduler, time, on_wall_clock)`
   take a `datetime`; `is_scheduled(scheduler)` returns `bool`;
   `next_scheduled_time(scheduler)` returns `datetime`. Only a real-time
   executor accepts wall-clock alarms; a due alarm fires on the next
   evaluatable cycle. Tags are not exposed.

3. **`scheduled()` is a handler selector.** It is valid only in a
   function-level `when` condition of a function that injects `scheduler`,
   and is true when the current evaluation is the node's alarm firing.
   A source on the stateless `alarm` of
   [ADR 0015](0015-pull-sources.md) does not use it: it publishes from a
   plain `when`, since every evaluation is its wake-up. A handler with `scheduled()` at top level receives
   no implicit input activation subscriptions; its default requirement that
   all time-series inputs be valid is unchanged. An explicit `valid()` instead
   requires no input valid and leaves validity decisions to the body. Such a
   scheduled handler adds no input to the node's activation set. When no
   handler names an input, the set is empty: every input is passive and the
   node evaluates only when scheduled. Explicit `modified()` also selects no
   input subscriptions. If no handler contributes active inputs and no other
   usable driver is armed, the compiler should warn; injecting a scheduler
   without using it to start evaluation is insufficient. Intentional dormant
   sinks remain valid source. See [handler selectors](../../user-guide/functions.md#handler-selectors-and-validity-handling).

4. **A runtime function may have no temporal parameters when it injects
   `scheduler`.** It is a source: `start { schedule(scheduler, 0s) }` asks
   for evaluation in the starting cycle. Its default all-input validity
   requirement is vacuously satisfied, and its default input activation set is
   empty.
   [ADR 0015](0015-pull-sources.md) extends this decision: a source may
   instead inject the stateless `alarm`, or be a generator written with
   `yield`.

5. **`passivate(input)` and `activate(input)`** are runtime statements
   whose argument is a direct temporal parameter of the function. They are
   evaluation-only: `start` and `stop` have no temporal input access. A passive
   input still holds its value and validity; it just no longer activates the
   node. The node's static activation policy is unchanged; activity is a
   per-evaluation effect.

## Consequences

- Library operators must preserve their state and recovery contracts.
  `schedule` requires recordable progress and positive-delay validation in
  `start` ([ADR 0011](0011-cache-declarations.md#scheduler-recovery-contract)).
  `throttle`, `batch`, `gate`, `lag` and `window` need the buffered-delta and
  queue state contracts (MIG-005).
- The language model's open questions on bare handlers without temporal
  parameters and on the explicit empty activation set are closed by
  decisions 3 and 4. LIB-002 (collection startup) can now be spelled with
  `start { schedule(scheduler, 0s) }` and `when scheduled()`; closing it is
  library work, not language work.
- `scheduled`, `passivate` and `activate` are intrinsic names and cannot be
  used as identifiers.
- Output access inside `start` and `stop` remains rejected; scheduling from
  `start` is admitted because the scheduler is a capability, not the output.
