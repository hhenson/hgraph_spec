# ADR 0015: pull sources: the `alarm` injectable, `yield`, and `while`

Status: proposed; spelling agreed with the project owner on 2026-09-28.

## Context

[ADR 0010](0010-lifecycle-capabilities.md) made a source a runtime function
with no temporal parameters that injects `scheduler`, schedules itself in
`start`, and publishes in a `when scheduled()` handler. That is three hooks
to publish one value, and `scheduler` binds hgraph's `NodeScheduler`: a
per-node state slot of tagged pending events that checkpoints capture and
restore. A constant source neither needs nor wants that.

Two scheduling capabilities serve different purposes. The reliable scheduler
retains tagged alarms, supports cancellation and queries, and participates in
checkpoint recovery. The one-shot alarm keeps the earliest request without
recordable state; it is neither stored nor recovered (ADR 0011).

A pull source can be described as a sequence of `(time, value)` publications.
The generator form expresses that sequence without requiring an author to
write an explicit resume state machine.

## Decision

### 1. `inject alarm` binds the stateless scheduler

`alarm` is an injectable that maps to `SingleShotScheduler`. It offers
`schedule(alarm, delay)` and `schedule_at(alarm, time)`, both available in
`start` and during evaluation. Repeated requests keep the earliest time; a
later request never cancels or postpones an earlier one. There is no
`is_scheduled(alarm)`, no `next_scheduled_time(alarm)`, no tag, no `un_schedule`, and
no wall-clock argument; using any of them names `scheduler` in the
diagnostic.

`alarm` is admitted only in a function with no temporal parameters. In such
a function every evaluation is the alarm firing, so the source publishes
from a plain `when`: with no temporal inputs the handler's implicit selector
is empty, and it runs on every evaluation. `scheduled()` belongs to
`scheduler` and is not admitted with `alarm`; the diagnostic says to write
the plain `when`. A function with temporal parameters that also needs an
alarm uses `scheduler`, whose state is what tells an alarm evaluation from
an input tick.

Nothing about `alarm` is recorded. After a restore the node's `start` runs
again and re-arms whatever it arms; this is the reconstructible policy of
`cache` (ADR 0011), not the recovered policy of `scheduler`.

ADR 0010 decision 3 is unchanged: `scheduled()` is the `scheduler`
selector. ADR 0010 decision 4 becomes: a runtime function may have no
temporal parameters when it injects `scheduler` or `alarm`, or when it is a
generator (decision 3).

### 2. When to use which

| | `alarm` | `scheduler` |
| --- | --- | --- |
| Node state | none | pending events (tagged inside hgraph; tags are not exposed, ADR 0010) |
| After a restore | `start` re-arms; nothing is restored | pending alarms are restored |
| Requests | `schedule(alarm, delay)`, `schedule_at(alarm, time)`; earliest wins | `schedule(scheduler, delay)`, `schedule_at(scheduler, time)`, `is_scheduled(scheduler)`, `next_scheduled_time(scheduler)` |
| Wall clock | no | `schedule(scheduler, delay, true)` under a real-time executor |
| Admitted in | sources only | any runtime function |

A source injects one of the two, never both: it has one wake-up mechanism.
`scheduled()` is the `scheduler` handler's selector; an alarm source's
plain `when` is its handler, because every evaluation is its wake-up.

Reach for `alarm` first. Move to `scheduler` the moment the wake-up must
survive a restart, be queried, coexist with input ticks, or follow the wall
clock. The standard library's `schedule` operator stays on `scheduler`
because it promises recovery and wall-clock alarms; `nothing` moves to
`alarm`, and a constant source reduces to the alarm shape below (the
library's `const` itself is a consequence, not this decision).

```hgl
fn constant(const value: i64, const delay: duration = 0s) -> i64 {
    inject alarm
    start { schedule(alarm, delay) }
    when { return value }
}

impl fn nothing<T>() -> T {
    inject alarm
    when { }
}
```

### 3. `yield time: value` makes a function a generator source

```ebnf
yield_statement = "yield", expression, ":", expression;
```

The pair uses the spelling the test harness already uses for a timed
sequence element, `time: value`, so a source and the sequence a test
compares it with read alike. A runtime function containing `yield` is a
generator source:

- It has no temporal parameters and declares a result type. It may use
  `const` parameters, `clock`, `logger`, `let` and `var` locals, `if` and
  `while`. It may not declare `state` or `cache`, inject `out`, `scheduler`
  or `alarm`, or contain `when`, `start` or `stop`: `yield` owns the output
  and the scheduling. A `return` carries no value: `yield` publishes, and a
  bare `return` finishes the source. `for` is not admitted (see the open
  questions): its iteration owns the body, so a suspension inside it could
  not resume.
- A `yield` is a statement of the body, of a `while` block or of an `if`
  statement. It cannot sit inside a value (an `if` used as an expression,
  a lambda), where a backend has no resume point to jump to.
- The body starts running in the node's first evaluation, which the runtime
  requests at start. `yield t: v` with a `duration` `t` means `t` after the
  evaluation time at which the body is running; with a `datetime` it is
  absolute. This makes a periodic source a `yield` inside a loop. It differs
  from a timed sequence element in a test,
  whose duration key is an offset from the run's start.
- Every resolved yield time must be strictly greater than the previous
  resolved yield time in this generator invocation. Equal or decreasing
  times raise a node error, including skipped past absolute entries and
  yields reached after resumption. The first yield has no predecessor.
- A negative duration is an error; it never becomes a skipped past target.
  An absolute datetime earlier than the current evaluation time is skipped
  and the body continues. A target equal to the current evaluation time
  publishes `v` immediately and the body continues. A second publication at
  one time is an error.
- A later target parks `v`, schedules the node for that target, and suspends the body.
  When the node evaluates, it publishes `v` and resumes the body after the
  `yield`. A body that ends is finished: the node never evaluates again.
- Locals live across suspensions as node-local storage and are not
  recorded. After a restore the body restarts from its first statement,
  which is what hgraph's generator does.

#### Operand evaluation, resolution and retention

1. Check t as `duration` or `datetime` and v against the output context.
   Evaluate t once, then v once. A time-expression failure prevents v;
   a payload failure prevents target validation. Existing phase and capability
   restrictions apply.
2. After both operands succeed, reject a negative duration with
   `yield.negative_duration`. Otherwise resolve
   a duration by checked addition to the current evaluation time; a datetime
   is already absolute. Arithmetic inside t belongs to step 1. Zero duration
   and negative scalar payloads are allowed, subject to step 3.
3. Require the target to be strictly greater than the preceding yield target
   in this invocation. Equal or earlier targets raise `yield.non_increasing_time`. The first
   yield has no predecessor. Skipped past targets count; resumption preserves
   the preceding target, while a fresh invocation starts without one.
4. After validation: a past absolute target skips publication and continues;
   a due target publishes and continues; a future target independently retains
   v, schedules the target and suspends. Skipping needs no parked copy.
5. On resumption, publish the retained payload and continue after the yield.
   Do not reevaluate operands or resolve the target again. Later source
   changes cannot change the retained payload. Engine bounds govern execution.
6. Operand, admission, arithmetic, ordering and retention failures propagate
   as node errors. Do not schedule, suspend, publish or continue that yield
   after failure. Preserve earlier completed effects and publications; no
   rollback is implied. This adds no effects, injectables or checkpoint rules.

See [compiler cases](../../../../compiler/cases_generator_operands.md) and
[examples](../../../examples/generator-yield-operands.hgl).
Audit evidence: [negative durations](https://github.com/hhenson/hgraph_spec_audit/blob/9967125fe6fad385e2d120e40049348402e0c686/runtime/validation/generator_negative/README.md)
and [target ordering](https://github.com/hhenson/hgraph_spec_audit/blob/9967125fe6fad385e2d120e40049348402e0c686/runtime/validation/generator_ordering/README.md).

```hgl
fn constant(const value: i64, const delay: duration = 0s) -> i64 {
    yield delay: value
}

fn heartbeat(const period: duration, const beats: i64) -> bool {
    var count: i64 = 0
    while count < beats {
        yield period: true
        count += 1
    }
}
```

A generator needs no runtime node kind. The compiler lowers the body to a
state machine: a resume position and the hoisted locals in node-local
storage, one parked value, and the stateless scheduler for the wake-up. A
generator is therefore sugar over decision 1, and the only runtime change
both decisions need is exposing `SingleShotScheduler` to HGL.

### 4. `while` is the conditional loop

```ebnf
while_statement = "while", [ expression ], block;
```

`while condition { ... }` repeats the block while the condition holds.
Omitting the condition means the loop is unbounded, as omitting a `when`
predicate means the default handler; there is no separate `loop` keyword.
`while` follows the iteration phase rule: it is a runtime statement. In a
composition body it is rejected, because a composition loop wires structure
rather than executes. There is no `break` or `continue` in this slice; a
loop ends when its condition is false or the function returns.

### 5. Reserved words

`yield` and `while` are hard reserved words. `alarm` is an injectable name
and, like `out`, `clock` and `scheduler`, stays contextual.

## Consequences

- `nothing` and constant sources can use the one-shot alarm contract.
- The library constant source is named `const`, never `const_`. A
  `const(f)` call naming a function remains the value-role selector; other
  `const(...)` calls resolve to the operator in scope. Its contract keeps an
  independent scalar input type and temporal output shape, with `delay` after
  them; see [MIG-009](../migration-requirements.md#mig-009-source-names-versus-native-identities).
- Evaluation of a source needs a finite bound. A generator may never finish;
  source completion alone cannot bound every harness run. The explicit end
  spelling remains a separate harness decision.

## Possible realization

An implementation could represent a generator by its resume position, live
locals and a pending publication. At a future yield it saves that state and
schedules resumption; at a due yield it publishes and continues in source
order. It must preserve skipped past absolute times, negative-duration and
duplicate-time errors, loops, conditionals and termination. Native coroutines and explicit state machines
are possible realizations, not language requirements.

## Open questions

- Whether a generator may carry a `stop` block. hgraph's `@generator.stop`
  exists; the block would run at teardown with the `const` parameters only.
- Whether an absolute `yield` time in the past should be a diagnostic
  rather than skipped. hgraph skips silently; this record follows it.
- `for` inside a generator, which hgraph's replay generators use over a
  constant collection. It needs the loop index hoisted beside the locals
  and an index-based loop in each backend; until then the checker rejects
  it and points to `while`.
- `break` and `continue` for `while`.
