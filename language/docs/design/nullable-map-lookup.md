# Nullable ordinary and atomic dictionary lookup

Status: accepted, 2026-10-14.

1. **LOOKUP-1 — Values.** During ordinary value evaluation, including
   value functions, node handlers and test bodies, ordinary dictionary values
   and `atomic<map<K, V>>` payloads use point-in-time value
   lookup. `m[key]` and `at(m, key)` return the present ordinary V and fail with
   `value.missing_key` on absence. Executed calls preserve that code through
   graph propagation; required constant evaluation rejects a missing key.
   Graph selection of a non-atomic temporal map remains connection selection.
2. **LOOKUP-2 — Safe absence.** `get(m, key)` and
   `get(m, key, default: null)` return the present ordinary V or `null` on
   absence. An explicit non-null `default: value` must have the map's exact V
   type; the result is ordinary V, including when the stored value is zero,
   false, or empty. Absence does not fail, publish, or create an empty REF.
   All written arguments are evaluated once in written order, including an
   explicit fallback even when the key is present; no lazy-default rule is
   introduced. A missing lookup alone does not suppress the containing handler:
   only that handler's explicit control/output operations determine publication.
3. **LOOKUP-3 — Presence checks.** With the null default, the result may
   initialize an inferred immutable `let`. Before payload use, establish its
   presence by comparing that binding with `null`. The
   [existing presence-refinement rules](nullable-replay-indexing.md#contextual-nullable-result-and-flow-refinement)
   apply: direct comparisons, negation, short-circuit flow, terminating
   branches and merges preserve facts; aliases and repeated reads require
   their own checks. Unchecked arithmetic, projection, argument passing,
   output assignment or return is rejected as `nullable.required_presence`.
4. **LOOKUP-4 — Scope.** A nullable local cannot escape through mutable
   locals, state, cache, closures, collections, fields, ordinary parameters or
   declared results. A proven-present V follows its existing ownership and
   borrowing rules. This adds no general nullable type annotation, truthiness,
   implicit null propagation or null-return suppression. Safe lookup of a
   present invalid borrowed temporal child remains open.

```hgl
const fn price_or_zero(book: map<str, i64>, key: str) -> i64 {
    let price = get(book, key)
    if price == null { return 0 }
    return price
}
```

| Operation | Absent key | Present key holding 0 |
|---|---|---|
| `book[key]`, `at(book, key)` | `value.missing_key` failure | 0 |
| `get(book, key)` | Successful `null` | Present 0 |
| `get(book, key, default: 5)` | 5, ordinary i64 | 0, ordinary i64 |

`_` remains an eval harness slot with no publication. A nullable lookup result
is an evaluation-local value observation, not such a slot. See
[source examples](../../examples/nullable-map-lookup.hgl),
[checking cases](../../../compiler/cases_nullable_map_lookup.md) and the
[native surface record](native-surface-proposal.md).
