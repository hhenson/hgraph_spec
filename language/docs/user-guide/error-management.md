# Absence and recoverable failures

Safe dictionary lookup returns absence as `null`. Bind it with `let` and prove
presence before using its value:

```hgl
const fn price_or_zero(book: dict<str, i64>, key: str) -> i64 {
    let price = get(book, key)
    if price == null { return 0 }
    return price
}
```

An exact non-null fallback, such as `get(book, key, default: 0)`, yields an
ordinary i64. Strict `book[key]` and `at(book, key)` fail with
`value.missing_key` on absence. These are ordinary/atomic value lookups;
borrowed temporal children retain their separate access rules. See
[nullable lookup](../design/nullable-map-lookup.md).

Use `result<T, E>` for local recovery. `ok(value)` and `err(error_value)`
construct its tags; the expected result type fixes both payload types.
Check `is_ok(local)` before reading `.value`, or `is_err(local)` before
reading `.error`. A result time series publishes one complete atomic outcome.
The intrinsic envelope needs no extra atomic wrapper. Graph uses of `is_ok`
and `is_err` follow ordinary automatic lifting and prove no input validity or
graph-side payload facts.

```hgl
native const fn parse_price(text: str) -> result<i64, error>

const fn doubled(text: str) -> result<i64, error> {
    let price = parse_price(text)?
    return ok(price * 2)
}

fn required_price(text: str) -> i64 {
    when { return parse_price(text)? }
}
```

In a compatible result-returning helper, postfix `?` unwraps success and
immediately returns the same failure. In a node `when`, failed
`result<T, error>` instead ends evaluation through its failure channel;
the normal output retains its declared type. Returning Err from a result-typed
node publishes data. Earlier writes stand if a later operation fails.

`error(message: "missing price", code: "value.missing_key")` constructs a
canonical diagnostic. The code may be omitted; `.message` is text and `.code`
requires a null presence check before payload use. On graph failure,
`.context` records the original node, phase and evaluation time; bind and
presence-check it before field access. Propagation/capture preserve that
context. A code outside the execution catalogue cannot satisfy `assert raises`.
Recoverable native functions declare `result<T, error>`. There is no source
`throw`, `try` or `catch` control. See the
[complete result and propagation rules](../design/result-errors.md).

An ordinary-T native operation can still fail under its strict contract.
Returning a result describes recoverable outcomes; it does not turn invalid
arguments, ownership violations or strict operation failures into Err data.

In graph composition, `let attempt = capture(required_price, text)` exposes
the normal connection as `attempt.value` and an error stream as `attempt.error`.
The closest capture consumes an evaluation failure once. Later cycles and
unrelated scheduled work continue; completed writes stand. Capturing a graph
covers failures leaving its whole subgraph, while capture inside a keyed child
body keeps errors with that child's key and lifetime.

```hgl
fn failure_this_cycle(failure: error, step: i64) -> bool {
    when modified(step) && valid(step) { return modified(failure) }
}

fn price_failures(text: str, step: i64) -> bool {
    let attempt = capture(required_price, text)
    failure_this_cycle(attempt.error, step)
}
```

`valid(attempt.error)` means a past failure remains readable;
`modified(attempt.error)` means a failure occurred in this cycle. Each failure
ticks, even with an equal message/code. Success leaves the retained error
unchanged. Start/stop failures escape capture. Match escaping catalogued
execution failures with `assert raises` after cleanup; inspect returned results
and captured errors with ordinary assertions. See
[capture rules and cases](../design/error-capture.md).
