# Graph capture and retained error streams

Status: accepted, 2026-10-14.

1. **CAPTURE-1 — Source boundary.** In graph composition,
   `let observed = capture(callable, args...)` composes that temporal callable
   under an error-capturing boundary. Ordinary graph-call selection, including
   permitted value-function lifting, applies. The inferred connection aggregate exposes
   `.value` with the callable's unchanged normal output type and `.error` as
   an `error` time series. For an outputless callable only `.error` exists.
   Callable/argument checking and written argument evaluation order are those
   of an ordinary call. Capture is not a value helper, error constructor or
   general source exception block. Capturing a node covers its evaluation;
   capturing a graph covers execution failures leaving that entire subgraph.
   The target must resolve to a temporal call; wiring-time scalar evaluation
   is not captured.
   Observing either connection retains the captured callable's required
   producer dependencies under ordinary graph pruning.
2. **CAPTURE-2 — Events and retention.** A captured evaluation failure
   publishes exactly one canonical error, preserving its original diagnostic
   and provenance. Before any failure `.error` is invalid. Each failure ticks,
   even with equal diagnostics. `.error` retains the last failure across
   silence and later success: `valid` means a failure has been retained,
   `modified` means a failure occurred this cycle. Success never clears it or
   republishes a stale error. Normal `.value` retains its ordinary data rules;
   absent success data is not fabricated as null or Err.
3. **CAPTURE-3 — Continue without rollback.** Under NOD-19/29, a captured
   node failure ends that evaluation; later scheduled nodes outside the
   capture boundary still run in rank order in the same cycle. Failure leaving
   a captured graph stops the remaining pass inside that graph for that cycle;
   unrelated outer work continues. The same graph/child remains live and
   runnable in later cycles. Prior writes stand, including normal output
   written before the failure. Capture adds no rollback, immediate retry,
   teardown or replay of the failed pass. Existing graph scheduling rules
   govern future wakes; capture must preserve future scheduled work.
   Work due only in the failed pass is not replayed in a later cycle as a retry.
   See GRF-11/14/16/22 and the node scheduler contract for ordinary future wakes.
4. **CAPTURE-4 — Nesting and keyed lifetime.** The closest capturing
   boundary consumes a failure once; an outer capture does not duplicate it.
   A new failure in a consumer is a separate event, even when `?` propagates
   a retained error whose original provenance is unchanged. Capture inside an existing
   [independent keyed child-composition body](iteration.md#dynamic-structures-independent-bodies-first)
   preserves that child's key and lifetime.
   Its failure does not prevent sibling children evaluating in that cycle;
   its error output/entry leaves with the child, under NOD-29. This adds no
   new keyed iteration syntax or ordinary reference storage.
5. **CAPTURE-5 — Escaping failures and tests.** Start/stop failures are
   uncapturable (NOD-20), including child creation/removal driven during
   evaluation. Eligibility follows the failing operation, not a retained
   diagnostic's historical `.context.phase`. A new `?` failure in `when` is
   an evaluation failure even when its error already has older provenance.
   Uncaptured execution failure leaves the graph under
   its existing cleanup contract. `assert raises("catalogued.code")` matches
   only an escaping execution failure after successful ordinary cleanup,
   preserving its code. Captured errors and returned Err values are data for
   ordinary assertions. Source/build failure, crash, timeout, assertion failure
   and cleanup failure cannot satisfy an expected execution error.

```hgl
fn parse_observed(text: str) {
    let observed = capture(required_price, text)
    consume_value(observed.value)
    consume_error(observed.error)
}
```

The example assumes ordinary typed consumer functions and the result helper
from the [result contract](result-errors.md). Both connections keep their own
validity and tick history. No allocation, error-message formatting cost or
native representation is promised by these source rules.

| Failure location | Observable boundary behavior |
|---|---|
| Captured node evaluation | Error ticks; earlier normal writes stand; unrelated work continues. |
| Later cycle succeeds | Value may tick; error remains valid and unmodified. |
| Equal failure on another cycle | Error ticks again. |
| Captured inner graph, itself inside capture | Inner error ticks once; outer error stays unchanged. |
| Keyed child evaluation | Error belongs to that key; siblings continue; removal ends its lifetime. |
| Start or stop | Failure escapes capture and requires normal cleanup/reporting. |

See [source examples](../../examples/error-capture.hgl),
[capture cases](../../../compiler/cases_error_capture.md),
[execution-error assertions](execution-error-assertions.md), and
[runtime failure rules](../../../runtime/node.md#failure).
