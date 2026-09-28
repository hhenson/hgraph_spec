# ADR 0015: pull sources: the `alarm` injectable, `yield`, and `while`

Status: proposed. The spelling was agreed with the project owner on
2026-09-28; nothing below is implemented in a compiler yet. Implementation
status will be recorded here as it lands.

## Context

[ADR 0010](0010-lifecycle-capabilities.md) made a source a runtime function
with no temporal parameters that injects `scheduler`, schedules itself in
`start`, and publishes in a `when scheduled()` handler. That is three hooks
to publish one value, and `scheduler` binds hgraph's `NodeScheduler`: a
per-node state slot of tagged pending events that checkpoints capture and
restore. A constant source neither needs nor wants that.

hgraph has two schedulers. `NodeScheduler` is the reliable one: tagged
alarms, cancellation, `is_scheduled`, wall-clock alarms, checkpoint capture
and restore. `SingleShotScheduler` is the stateless one: it marks the node
to evaluate now, after a delay or at a time, keeps the earliest request,
holds no state and is neither stored nor recovered (ADR 0011). It exists
only for C++ nodes today.

hgraph's own pull sources (`const`, `nothing`, `replay`, `replay_const`) are
Python generators: a function yields `(time, value)` pairs and the runtime
publishes each pair at its time. Nothing in HGL expresses that shape.

## Decision

### 1. `inject alarm` binds the stateless scheduler

`alarm` is an injectable that maps to `SingleShotScheduler`. It offers
`alarm.schedule(delay)` and `alarm.schedule_at(time)`, both available in
`start` and during evaluation. Repeated requests keep the earliest time; a
later request never cancels or postpones an earlier one. There is no
`is_scheduled()`, no `next_scheduled_time()`, no tag, no `un_schedule`, and
no wall-clock argument; using any of them names `scheduler` in the
diagnostic.

`alarm` is admitted only in a function with no temporal parameters. In such
a function every evaluation is the alarm firing, so `scheduled()` is true on
every evaluation and needs no stored answer. A function with temporal
parameters that also needs an alarm uses `scheduler`, whose state is what
tells an alarm evaluation from an input tick.

Nothing about `alarm` is recorded. After a restore the node's `start` runs
again and re-arms whatever it arms; this is the reconstructible policy of
`cache` (ADR 0011), not the recovered policy of `scheduler`.

ADR 0010 decision 4 becomes: a runtime function may have no temporal
parameters when it injects `scheduler` or `alarm`, or when it is a
generator (decision 3).

### 2. When to use which

| | `alarm` | `scheduler` |
| --- | --- | --- |
| Node state | none | pending events and their tags |
| After a restore | `start` re-arms; nothing is restored | pending alarms are restored |
| Requests | `schedule(delay)`, `schedule_at(time)`; earliest wins | the same, plus tags, `un_schedule`, `is_scheduled`, `next_scheduled_time` |
| Wall clock | no | `schedule(delay, true)` under a real-time executor |
| Admitted in | sources only | any runtime function |

Reach for `alarm` first. Move to `scheduler` the moment the wake-up must
survive a restart, be cancelled or replaced, coexist with input ticks, or
follow the wall clock. The standard library's `schedule` operator stays on
`scheduler` because it promises recovery and wall-clock alarms; `const_`
and `nothing` move to `alarm` or to a generator.

```hgl
impl fn const_<T>(const value: T, const delay: duration = 0s) -> T {
    inject alarm
    start { alarm.schedule(delay) }
    when scheduled() { return value }
}

impl fn nothing<T>() -> T {
    inject alarm
    when scheduled() { }
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
  `const` parameters, `clock`, `logger`, `let` and `var` locals, `if`,
  `for` and `while`. It may not declare `state` or `cache`, inject `out`,
  `scheduler` or `alarm`, or contain `when`, `start` or `stop`: `yield`
  owns the output and the scheduling.
- The body starts running in the node's first evaluation, which the runtime
  requests at start. `yield t: v` with a `duration` `t` means `t` after the
  time at which the body is running; with a `datetime` it is absolute. This
  is hgraph's generator rule, and it is what makes a periodic source a
  `yield` inside a loop. It differs from a timed sequence element in a test,
  whose duration key is an offset from the run's start.
- An absolute time earlier than now is skipped and the body continues. A
  time equal to now publishes `v` immediately and the body continues. A
  second publication at one time is an error, as in hgraph.
- A later time parks `v`, schedules the node for `t`, and suspends the body.
  When the node evaluates, it publishes `v` and resumes the body after the
  `yield`. A body that ends is finished: the node never evaluates again.
- Locals live across suspensions as node-local storage and are not
  recorded. After a restore the body restarts from its first statement,
  which is what hgraph's generator does.

```hgl
impl fn const_<T>(const value: T, const delay: duration = 0s) -> T {
    yield delay: value
}

impl fn replay_const<T>(const values: list<tuple<datetime, T>>) -> T {
    for item in values {
        yield item[0]: item[1]
    }
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

`yield` and `while` join the hard reserved words in the keyword table of
`token.cpp` and in the developer guide. Neither is used as an identifier in
the standard library, the examples or the compiler tests. `alarm` is an
injectable name and, like `out`, `clock` and `scheduler`, stays contextual.

## Consequences

- `const_` and `nothing` in the standard library are re-expressed as above
  once a compiler implements this record; the catalogue entries for `const`
  and `nothing` then move from native-provider to implemented, and
  `replay_const` becomes authorable.
- The evaluation harness needs an end for a source: today `eval` requires a
  time-series input to bound the run. A generator ends by itself, so `eval`
  may run a source until it finishes; a source on `alarm` still needs an
  explicit end. The harness spelling for that end remains open in
  "Tests and the evaluation harness".
- Compiler work: the `alarm` injectable (hgraph already has an argument
  provider for `SingleShotScheduler`), the `while` statement in runtime
  bodies, the generator classification and lowering, and the two keywords.
  The Rust compiler mirrors the same source. Editor tooling adds the two
  keywords.

## Implementation

`yield` is a lowering, not a runtime feature. A generator becomes the same
static node a `when scheduled()` source is today: node-local storage, an
`eval` hook and a wake-up. hgraph already has every runtime piece: the
argument provider for `SingleShotScheduler` in `eval`, the
`schedule_on_start` node attribute, and the generated cache struct in one
`State<>` slot (ADR 0011). The runtime does not change; the compiler does.

1. **Syntax.** `yield` and `while` in the keyword table, two productions in
   the declarative grammar, and two typed-HIR statements beside `when` and
   `for`: a yield with a time and a value, and a while with an optional
   condition.
2. **Classification.** A body containing `yield` is a runtime function of a
   new kind, generator. The checker enforces decision 3: no temporal
   parameters, a result type, the time typed `duration` or `datetime`, the
   value typed as the result, and none of `state`, `cache`, `out`,
   `scheduler`, `alarm`, `when`, `start` or `stop`. `while` is admitted in
   runtime bodies only.
3. **Lowering.** A pass between typed HIR and hgraph IR rewrites the
   generator into ordinary runtime-node IR. The body is split into blocks
   at every `yield`, and the yield points are numbered, with 0 the entry and
   one more value meaning finished. Every local live across a suspension is
   hoisted into the cache struct together with the resume index, the parked
   value and, for a `for` over a `const` collection, the loop index. The
   node is marked `schedule_on_start`, so its first evaluation needs no
   `start` hook. The generated `eval` publishes a parked value first (after
   the duplicate-time check against the output's last modified time),
   dispatches on the resume index, and runs to the next yield: an earlier
   absolute time is skipped, a time equal to now publishes and continues, a
   later time parks the value, requests `alarm.schedule_at(instant)`, stores
   the resume index and returns. Falling off the end marks the node finished.
4. **Emission.** The C++ emitter gains the `alarm` parameter
   (`hgraph::SingleShotScheduler`), a plain `while`, and the dispatch: a
   `switch` on the resume index that jumps to a label at each yield point.
   Jumping into nested blocks is legal because hoisting leaves no local with
   an initializer in the `eval` body, the same technique as C# iterators and
   stackless coroutine libraries. The Rust compiler consumes the same
   lowered IR and, having no `goto`, emits a loop over a `match` on the
   resume index.
5. **Order.** Keywords, parser and HIR with parser tests; `alarm` end to
   end, which already lets `const_` and `nothing` drop the state slot;
   `while` in runtime bodies; the generator classification, lowering and
   dispatch with tests for a constant, a heartbeat, a replay over a `const`
   list, a skipped past time, a same-time publication and the duplicate-time
   error; then the standard-library re-expression and the editor keywords.

Why not C++20 coroutines: the frame is heap-allocated at start, suspension
and exceptions are their own model, there is no Rust mirror, and hgraph
has one runtime model with no second node kind. The per-tick cost of the
state machine is a switch, a publication and one `schedule_node` call, with
no allocation and no scheduler state slot.

ADR 0011 refuses sources with caches under the component checkpoint
contract, so a generator inside a checkpointed component is refused until
that contract admits reconstructible storage on sources.

## Open questions

- Whether a generator may carry a `stop` block. hgraph's `@generator.stop`
  exists; the block would run at teardown with the `const` parameters only.
- Whether an absolute `yield` time in the past should be a diagnostic
  rather than skipped. hgraph skips silently; this record follows it.
- `break` and `continue` for `while`.
