# Capability access in HGL source

Status: source-syntax clarification, 2026-09-30.

HGL reads [clock observations](clock-properties.md) through read-only
properties. Ordinary sequence elements use indexing. Capability actions and the
other specified queries use ordinary function-call syntax with the
capability as the first argument. There are no capability method calls or
method-call aliases. A dot selects a property or field where admitted;
it does not supply an implicit receiver to a call.

In a capability function call, the first argument is a direct name introduced
by `inject`.
It is an approved borrowing position of the selected capability operation,
not an ordinary first-class value. This rule does not permit assigning,
returning, storing or passing a capability to arbitrary user functions.
Existing capability availability, phases, ownership and failure rules remain
unchanged. Native implementation interfaces do not prescribe HGL spelling.

| Capability | Access form | Source forms |
|---|---|---|
| `clock` | Read-only properties, each `datetime` | `clock.evaluation_time`, `clock.now`, `clock.next_cycle_evaluation_time` |
| `scheduler` | Receiver-first functions | `schedule(scheduler, delay)`, `schedule(scheduler, delay, on_wall_clock)`, `schedule_at(scheduler, time)`, `schedule_at(scheduler, time, on_wall_clock)`, `is_scheduled(scheduler)`, `next_scheduled_time(scheduler)` |
| `alarm` | Receiver-first functions | `schedule(alarm, delay)`, `schedule_at(alarm, time)` |
| `logger` | Receiver-first functions | `info(logger, value)` for the existing informational logging operation |
| `global_state` | Receiver-first functions | `get(global_state, key)`, `set(global_state, key, value)` |

These function names are prelude names, not new reserved words. The direct
capability operand distinguishes these forms from ordinary graph operators
with the same name, such as the standard `schedule` operator. No ordinary
overload may receive the capability as a first-class value. Existing named
argument binding applies to non-capability parameters; the direct capability
operand occupies the first positional argument.

`len(result)` and `result[index]` are ordinary value operations, not
capability access. [Nullable sequence indexing](nullable-replay-indexing.md)
requires a presence guard before a nullable payload is used; the general
sequence source-type dependencies remain open. Run-wide get/set, contextual
result typing and errors are specified in
[ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md). The key is a
const string expression, bound with one exact ordinary type before start;
hooks use that prepared entry without key lookup or type dispatch. A bound
entry without a value fails get, distinct from an absent sequence element.
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
