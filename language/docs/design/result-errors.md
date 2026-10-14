# Tagged results, canonical errors and propagation

Status: accepted, 2026-10-14.

1. **RESULT-1 — One outcome.** `result<T, E>` is an ordinary tagged value:
   success owns one T, failure owns one E, and the other payload is absent.
   T and E must be ordinary value types, not temporal endpoints, REF or SIGNAL.
   A temporal `result<T, E>` carries one complete atomic outcome per
   publication, never a tuple or bundle with independently valid fields.
   Equal outcomes still tick; silence is not an outcome. Finite eval/publication
   profiles admit it when both payloads are admitted ordinary atomic payloads,
   additionally admitting `error` as an atomic payload.
2. **RESULT-2 — Construction.** `ok(value)` constructs success and
   `err(value)` constructs failure. Both type arguments come from the expected
   `result<T, E>` context, checking the supplied payload against its side.
   An unconstrained `let x = ok(1)` is rejected as `result.context_type`.
   Ordinary ownership/storage rules apply. Equality of the same result type
   requires equality of both payload types: compare tags, then the selected
   payload. This adds no ordering or hashing.
3. **RESULT-3 — Checked access.** `is_ok(value)` and `is_err(value)` are
   ordinary Boolean predicates. For an immutable `let` or ordinary parameter
   binding holding a result,
   `is_ok(local)` proves success and `is_err(local)` proves failure; their false
   branches prove the other tag. `.value` requires success and `.error`
   requires failure, otherwise reject `result.required_variant`. Negation,
   short-circuit flow, terminating branches, aliases and merges follow the
   [nullable refinement discipline](nullable-replay-indexing.md#contextual-nullable-result-and-flow-refinement).
   A saved bool or fresh result expression establishes no local payload fact.
   A mutable `var` does not receive a retained tag fact; bind an immutable
   snapshot instead. In node evaluation, the temporal input's ordinary
   payload read must first meet its existing validity/readability rules.
   A tag predicate establishes no time-series validity or modification fact.
4. **ERROR-1 — Canonical diagnostic.** `error(message: "text")` constructs
   an ordinary atomic error; optional `code: "value.missing_key"` supplies
   its code. `.message` is str. `.code` is a contextual nullable str: bind it
   with `let` and prove presence before payload use, under LOOKUP-3/4.
   Omission means no code, not an invented unknown-code alias. Domain codes
   outside the [execution catalogue](error-catalogue.md#execution-errors)
   cannot satisfy `assert raises`. Known execution codes must be preserved.
   Error equality compares code presence/content and message, excluding
   provenance. Equality of diagnostics never deduplicates failure events.
5. **ERROR-2 — Provenance.** `.context` is a read-only contextual nullable
   provenance record. A locally constructed error has no context. On first
   entry into a graph failure channel, the graph attaches the failing node's
   identity (`.node: str`), phase (`.phase: str`, one of `"evaluation"`,
   `"start"`, `"stop"`), and logical time (`.evaluation_time: datetime`).
   Node identity identifies the original node/graph instance in that run;
   its display spelling is not a stable error code. Bind and presence-check
   context before field access. Propagation and nested capture preserve the
   original code, message and context rather than replacing them with the
   catching node. Provenance is owned by the error's value lifetime.
6. **PROP-1 — Value-function propagation.** Postfix `expression?` evaluates
   its `result<T, E>` operand once. Success yields T; failure immediately ends
   the current value function by returning `err` with that same E payload.
   The declared result must be `result<U, E>` with the same resolved E,
   otherwise reject `result.propagation_type`. Completed effects stand.
   Strict execution failure is not implicitly converted to result data.
7. **PROP-2 — Handler propagation.** In a node `when`, `?` accepts only
   `result<T, error>`. Success yields T; failure ends the handler/evaluation
   through the implicit node failure channel with that canonical error.
   The normal output retains its declared type. A failing return expression
   writes no success value; earlier output/state/effect writes stand without
   rollback. `return err(...)` from a result-typed handler instead publishes
   complete ordinary result data and does not fail the node.
8. **PROP-3 — Syntax and native contracts.** `?` has postfix call/index/field
   precedence and cannot cross a newline. `a()? + b` propagates before addition.
   It is admitted in ordinary value functions and node `when`, not graph
   composition, direct start/stop handlers or generator bodies
   (`result.propagation_phase`). It does not unwrap null. Native functions
   offering recoverable failure declare `result<T, error>`; callers inspect
   or propagate it. No `throws` clause or `throw`/`try`/`catch` source control
   is added. Strict operations retain their execution-failure contracts.

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

| Operation | Failure handling |
|---|---|
| Guard result and read `.error` | Ordinary local recovery/data inspection. |
| `?` in result-returning helper | Return the same failure payload. |
| `?` in node `when` | Node execution failure, optionally captured. |
| Return Err as normal result output | Complete tagged data publication. |
| Finish handler without writing | No publication. |
| Nullable lookup returns null | Check absence; neither Err nor automatic failure. |

See [source examples](../../examples/result-errors.hgl),
[checking cases](../../../compiler/cases_result_errors.md) and
[rejection fixtures](../../examples/reject/result-errors.hgl).
