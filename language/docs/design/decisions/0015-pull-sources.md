# ADR 0015: pull sources: the `alarm` injectable, `yield`, and `while`

Status: proposed. The spelling was agreed with the project owner on
2026-09-28. The C++ compiler implements decisions 1 to 5 in hgraph PR
[#1670](https://github.com/hhenson/hgraph/pull/1670) (open); the Rust
compiler has not started. The standard-library re-expression and the
`const` spelling (consequences) are follow-ups.

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
| Requests | `schedule(delay)`, `schedule_at(time)`; earliest wins | the same, plus `is_scheduled()` and `next_scheduled_time()` |
| Wall clock | no | `schedule(delay, true)` under a real-time executor |
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
    start { alarm.schedule(delay) }
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

`yield` and `while` join the hard reserved words in the keyword table of
`token.cpp` and in the developer guide. Neither is used as an identifier in
the standard library, the examples or the compiler tests. `alarm` is an
injectable name and, like `out`, `clock` and `scheduler`, stays contextual.

## Consequences

- `nothing` in the standard library is re-expressed on `alarm` (hgraph_std
  PR [#5](https://github.com/hhenson/hgraph_std/pull/5)), and its catalogue
  entry moves from native-provider to implemented.
- hgraph's `const` reduces to the alarm shape above, but its library
  spelling is `const`, never `const_` (project owner, 2026-09-29: a
  function is not the `const` modifier, and the replacement of a native
  identity keeps its name), and its contract keeps
  [MIG-009](../migration-requirements.md#mig-009-source-names-versus-native-identities):
  an independent scalar `T` and output shape `S`, with `delay` after them.
  hgraph PR [#1671](https://github.com/hhenson/hgraph/pull/1671) supplies
  the mapping: `const` is admitted as an operator, function, instantiation
  and imported name, a `const(f)` call whose one argument names a function
  stays the value-role selector of ADR 0008, and any other `const(...)`
  call goes to the operator in scope. hgraph_std PR #5 then declares
  `const` on the alarm; no `const_` is added.
- The evaluation harness needs an end for a source: today `eval` requires a
  time-series input to bound the run, and that stays the rule. A generator
  that reaches the end of its body finishes, but decision 4 admits
  `while { yield ... }`, which never does, so `eval` cannot run a source to
  completion as its bound. A harness spelling for an explicit end remains
  open in "Tests and the evaluation harness"; the example pairs each source
  with a sink on a ticking input.
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
3. **IR.** hgraph IR keeps `yield` and `while` as statements and marks the
   callable a generator. It has no labels or jumps, so the state machine is
   not an IR rewrite: each backend lowers the generator in its own emitter,
   where the target language's control flow is available.
4. **C++ emission.** The emitter numbers the yield points in body order,
   with `-1` meaning finished. One cache struct (the ADR 0011 slot) holds
   the resume index, a parked flag, the parked value and every local of the
   body, hoisted so that no C++ local with an initializer sits in the
   `eval` body: a `let` or `var` becomes an assignment to its field and
   every read goes through the slot. The node carries `schedule_on_start`,
   `start` rebuilds the struct, and `eval` takes the slot, the
   `SingleShotScheduler` and the output. `eval` first publishes a parked
   value, then dispatches: a `switch` on the resume index jumps to the label
   after the last yield. At a yield the instant is `alarm.now()` plus the
   duration, or the datetime itself; earlier than now is skipped; equal to
   now publishes, after a duplicate-time check against the output's
   modified flag, and the body continues; later parks the value, stores the
   resume index, schedules the alarm at the instant and returns. Falling
   off the end, or a bare `return`, stores `-1`. Jumping into a `while` or
   `if` block is legal C++ because hoisting leaves no initialization to
   bypass, the technique of C# iterators and stackless coroutine libraries.
   The Rust emitter, having no `goto`, lowers the same IR to a loop over a
   `match` on the resume index.
5. **Order.** Keywords, parser and HIR with parser tests; `alarm` end to
   end, which lets `nothing` and a constant source drop the state slot;
   `while` in runtime bodies; the generator classification and the C++
   state machine with tests for a constant, a counting loop, a single parked
   value, an absolute time, a skipped past time, a same-time publication,
   a bare return and a yield inside `if`, and the duplicate-time error;
   then the standard-library re-expression and the editor keywords. The
   C++ compiler implements all of this except the last two.

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
- `for` inside a generator, which hgraph's replay generators use over a
  constant collection. It needs the loop index hoisted beside the locals
  and an index-based loop in each backend; until then the checker rejects
  it and points to `while`.
- A scripted test cannot yet expect an error, so the duplicate-time rule is
  covered by the runtime error message alone.
- `break` and `continue` for `while`.
