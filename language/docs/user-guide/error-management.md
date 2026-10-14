# Absence and recoverable failures

Safe dictionary lookup returns absence as `null`. Bind it with `let` and prove
presence before using its value:

```hgl
const fn price_or_zero(book: map<str, i64>, key: str) -> i64 {
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
