# Capability functions in HGL source

Status: source-syntax clarification, 2026-09-30.

HGL invokes capability operations with ordinary function-call syntax and the
capability as the first argument. It does not have a capability method-call
form or method-call aliases. A dot selects a field where field projection is
admitted; it does not supply an implicit receiver to a call.

The first capability argument is a direct name introduced by `inject`.
It is an approved borrowing position of the selected capability operation,
not an ordinary first-class value. This rule does not permit assigning,
returning, storing or passing a capability to arbitrary user functions.
Existing capability availability, phases, ownership and failure rules remain
unchanged. Native implementation interfaces do not prescribe HGL spelling.

| Capability | Source forms |
|---|---|
| `clock` | `evaluation_time(clock)`, `now(clock)`, `next_cycle_evaluation_time(clock)` |
| `scheduler` | `schedule(scheduler, delay)`, `schedule(scheduler, delay, on_wall_clock)`, `schedule_at(scheduler, time)`, `schedule_at(scheduler, time, on_wall_clock)`, `is_scheduled(scheduler)`, `next_scheduled_time(scheduler)` |
| `alarm` | `schedule(alarm, delay)`, `schedule_at(alarm, time)` |
| `logger` | `info(logger, value)` for the existing informational logging operation |
| `replay_input` | `len(replay_input)`; slot access uses `replay_input[index]` |
| `capture` | `begin(capture)`, `append(capture, time, delta)` |

These operation names are prelude names, not new reserved words. The direct
capability operand distinguishes these forms from ordinary graph operators
with the same name, such as the standard `schedule` operator. No ordinary
overload may receive the capability as a first-class value. Existing named
argument binding applies to non-capability parameters; the direct capability
operand occupies the first positional argument.

`len` retains the language's length-operation spelling; admitting the
`replay_input` capability adds neither a `length` alias nor a new keyword.
[Indexed replay reads](nullable-replay-indexing.md) return a contextual nullable
payload and require a presence guard before payload use. The replay/capture
signatures and errors are in
[ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md).
Clock/scheduler semantics remain in
[ADR 0010](decisions/0010-lifecycle-capabilities.md), and the source-only
alarm remains in [ADR 0015](decisions/0015-pull-sources.md).

`scheduled()` is unchanged: it is a handler selector with no capability
argument, not a scheduler query. `passivate(input)` and `activate(input)`
retain their direct temporal-input argument. `delta_value(endpoint)` retains
its temporal endpoint argument. None becomes a method call.

```hgl
fn constant(const value: i64, const delay: duration = 0s) -> i64 {
    inject alarm
    start { schedule(alarm, delay) }
    when { return value }
}
```
