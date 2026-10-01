# Read-only clock properties

Status: source-syntax clarification, 2026-09-30.

`inject clock` supplies the node or admitted value helper with the evaluation
clock. The three HGL observations are read-only properties of that direct
injected binding:

| Property | Type | Meaning |
|---|---|---|
| `clock.evaluation_time` | `datetime` | Logical time of the current evaluation cycle. |
| `clock.now` | `datetime` | The engine's current estimate of wall-clock time. |
| `clock.next_cycle_evaluation_time` | `datetime` | Earliest possible following cycle: evaluation time plus the engine's smallest step. |

Use ordinary property selection with no parentheses. There are no receiver
method-call or free-function aliases for these observations. The property
names are contextual to the injected clock, not new reserved words or
global value bindings. Other objects' ordinary fields keep their existing
field-selection rules.

Reading a property produces an ordinary owned `datetime` value. It does not
publish, schedule, change the engine clock or make the clock a first-class
value. The capability retains its existing availability and borrowing rules,
including use from start, evaluation and stop and admitted value helpers.
Writing or compound-assigning a property is a checking error. A property
cannot be invoked as a function or used to retain the capability itself.

## Timing semantics

These properties retain the [engine clock rules](../../../runtime/execution_engine.md#state):

- Evaluation time is set before each cycle, is the same for every node in
  that cycle, and remains constant throughout it. Successive cycle times
  increase. At start, evaluation time is the run's start time.
- Next-cycle evaluation time is evaluation time plus the smallest engine
  step. It changes with evaluation time and is constant within a cycle.
  It is a lower bound on a following cycle's time, not a promise that a
  cycle will run then or a scheduling request.
- In real time, now reads UTC time from the computer's clock. In simulation,
  it is evaluation time plus the elapsed real-time lag of the current cycle.
  It may change between property reads within one node or one cycle; it is
  not a cycle-frozen sample. This contract adds no monotonicity guarantee for
  now and does not equate now with evaluation time in real time.

Read-only describes the caller's access, not a promise that all properties
remain constant. Saving a property in a local snapshots the value at that
read; later property reads do not change the saved datetime.

The runtime clock's lag remains outside the HGL property surface specified
here. This extension does not add `clock.lag`, a new clock provider, a new
lifecycle phase or new end-of-run timing semantics.

## Actions and queries

Clock properties do not change the spelling of capability actions or the
other admitted queries. Those keep
[receiver-first function syntax](capability-function-syntax.md), including
`schedule(alarm, delay)`, `schedule_at(scheduler, time)`,
`is_scheduled(scheduler)`, `get(global_state, key)` and
`set(global_state, key, value)`. For example:

```hgl
schedule_at(alarm, clock.next_cycle_evaluation_time)
```

The property is read to supply a datetime argument; the function performs
the scheduling action. The `scheduled()` handler selector remains unchanged.
