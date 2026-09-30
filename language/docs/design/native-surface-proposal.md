# Native surface completion

Status: accepted names and direction; unresolved contracts are identified below.

## Values are values

Type erasure is an implementation detail, not a separate HGL value category.
Reading a value uses ordinary HGL expressions and operations. A delta is obtained
with the existing name `delta_value`, regardless of whether its type is named,
generic, or known only through runtime metadata. Do not introduce `value_equals`,
erased-value accessors, or different spelling based on the C++ representation.

This decision does not claim that the compiler already lowers every value and
delta shape. Missing transport, borrowing, and operator implementations are
compiler/library work behind the same language surface.

The [scalar delta-value contract](delta-value-metadata.md) specifies runtime
admission and scalar typing for the sole accessor `delta_value` and gives
the generic explicit-delta pass-through fixture. The constructor remains
`delta<S>(...)`. Structural contextual delta transport/application remain separate.


## Membership and access

| Function | Contract |
| --- | --- |
| `key_set(value)` | The live set of map keys |
| `modified(key_set(value))` | Keys were added or removed, not merely child values changed |
| `contains(collection, key)` | Membership, independent of a child's validity; strings use substring membership |
| `at(value, key_or_index)` | Strict access; missing key or out-of-range index is an error |
| `get(value, key_or_index, default=null)` | Proposed safe lookup; nullable and invalid-child rules remain open |

The `default=null` notation above describes the parameter default. HGL named
call arguments retain their existing colon syntax: `get(value, key, default: 0)`.


```hgl
fn membership_changed(value: map<i64, f64>) -> bool {
    when modified(key_set(value)) { return true }
}

fn read_price(value: map<i64, f64>, key: i64) -> f64 {
    when { return at(value, key) }
}

fn key_changes(value: map<i64, f64>) -> i64 {
    when {
        let ks = key_set(value)
        var count = 0
        for key in elements(ks, added) { count += 1 }
        for key in elements(ks, removed) { count += 1 }
        return count
    }
}
```

`key_set` is borrowed within the evaluation; it does not copy keys into a new
container. Its added/removed ranges use the input's current projected delta,
not stale storage changes from an earlier evaluation. A local projection must
use `let`, not mutable `var`. The borrowed view must not escape the evaluation.
Returning it from a `when`, or assigning it to `out`, synchronously copies its
current contents into owned output storage. This aligns the whole set, including
removals; it does not return the borrowed projection itself. Structural children
read with `at` use the same complete-value copy path when written to an output.
The collection write rules determine the output delta, including empty deltas
when a complete value is written without changing membership.
An ordinary child-only tick is rejected by the membership timestamp without
scanning keys. A sampled reference rebind uses the input's projected key ranges.
`last_modified` on a runtime key-set projection is deliberately rejected until
the compiler can retain membership history across rebinds. A composition-level
`key_set` endpoint already supplies persistent tracking.


Open detail for `get`: whether a present but invalid child returns the fallback
or remains distinct from an absent key/index. The proposed rule is absent-only,
preserving removal versus invalidation; this detail still needs agreement.
The nullable-expression and result contract also remains to be specified. Neither
an invented zero value nor an implicit no-output tick implements nullable lookup.

## Windows

The accepted initial accessors are `at(window, index)`, `time_at(window, index)`,
`front(window)`, `back(window)`, and `removed_value(window)`. For tick-count windows, access preserves logical order through storage wraparound. Indices are
zero-based in oldest-to-newest logical order. Strict bounds errors propagate
through the generated node; they are not hidden inside a `noexcept` wrapper.
List `front` and `back` use the same strict bounds policy.
Strict list/map value access also rejects a present but invalid child instead
of reading its retained storage. `valid(at(value, key))` can inspect the child
without reading its payload, but the key/index must still exist.

```hgl
use hgraph.native as native

fn evicted(value: rolling<i64, 3, 1>) -> i64 {
    when {
        if native::has_removed_value(value) { return removed_value(value) }
    }
}
```

`removed_value` is the evaluation's evicted sample. Guard it with
`has_removed_value`; it is not an accessor for an arbitrary historical sample.
Existing native window metadata includes `capacity`, `period`, `min_period`,
`is_full`, `has_removed_value`, and `first_modified`.

## Remaining contracts

Native bindings for these operations must preserve value and borrowing
contracts, including dependent results and declared effects. A compiler
intrinsic is one possible realization, not a distinct source operation.

Further work, using the same ordinary value operations:

- General nullable `get`, including its invalid-child policy.
- Uniform current/delta-value expression lowering for atomic collections,
  tuples, structs and generic values; no representation-specific accessors.
- Atomic collection lookup and traversal, which must use scalar collection
  views rather than pretending they are TSL/TSS/TSD endpoints.
- Native bundle/reference patterns and duration-window const generics.
- Numeric, text and temporal value operations beyond existing arithmetic and
  predicates; preserve existing C++ semantics and flag genuinely undecided
  names or policies separately.

Storage pointers, slot indices, notifiers, binding mutation and ownership hooks
are not proposed HGL functions. Output mutations retain the already accepted
HGL surface and are not renamed as part of this work.
