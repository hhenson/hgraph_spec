# Live map key connections

Status: accepted, 2026-10-16.

`book[key]` selects a connection during graph composition when `book` is a
non-atomic `dict<K, V>`. It does not perform ordinary dictionary value lookup.

| Context | Result | Absent key |
|---|---|---|
| Graph selection from non-atomic `dict<K, V>` | `ref<V>` designating a child | Publish empty REF. |
| Ordinary map value or `atomic<dict<K, V>>` payload: `book[key]`, `at(book, key)` | Current ordinary V | `value.missing_key` failure. |
| Ordinary/atomic map value: `get(book, key)` | Guarded nullable local | Successful `null`. |
| Ordinary/atomic map value: `get(book, key, default: fallback)` | Ordinary V for an exact-V non-null fallback | The fallback. |

1. **MAP-CONN-1 — Selection.** A fixed K or a temporal K input selects one
   key. A readable new key retargets the connection to that key's child.
   Selection reads no child payload during wiring and stores no ordinary V.
   An ordinary V consumer follows the resulting REF under TS-14/17.
2. **MAP-CONN-2 — Membership.** With a valid, readable selected key, absence
   publishes empty REF and unbinds the ordinary consumer, including before
   the map's first publication. Later arrival selects the child even while
   the key is silent. Membership is distinct from child validity: a present
   invalid child is designated, not replaced by empty REF. Invalidating that
   child does not remove the key or change its designation. Its later valid
   publication reaches the ordinary consumer under the existing input rules.
3. **MAP-CONN-3 — Removal and lifetime.** Removing the selected key clears
   the selector's connection in the removal cycle. A separately held REF to
   that child remains readable through that cycle (TS-11), then expires at
   the next engine cycle (TS-23). Later-cycle reinsertion creates a new child
   identity; the selector selects it, while the old held REF stays expired.
   Same-cycle restoration retains the same child under TS-11.
4. **MAP-CONN-4 — Ticks and sampling.** First publication, including empty,
   and changed designation tick the REF. Repeated empty or equal designation
   does not tick or resample (TS-16). A changed valid target is sampled even
   when it holds the same payload as the previous target; sampling preserves
   its producer timestamp. Selected child publications reach ordinary
   consumers without a REF tick. Unselected siblings do not tick the REF or
   republish its selected value. A key change from one absent key to another
   repeats the equal empty designation.

Graph selection below `ref<dict<K, V>>` composes with existing WIR-5
reference-backed access: a changed outer map designation reselects the held
key in that map. It does not permit node evaluation to read beneath REF.

This initial selection profile covers V with no REF leaves. Existing
`dict<K, ref<T>>` and structural REF type formation and binding remain valid;
this scope is not a new type-formation prohibition. Reference-to-reference
identity still follows the runtime's flattening rule. Selection of REF-valued
or REF-containing children, including an unpublished inner REF, remains a
separate follow-up rather than an inferred nested `ref<ref<T>>` result.

Ordinary/scalar and atomic dictionary reads retain [LOOKUP-1–4](nullable-map-lookup.md),
including `value.missing_key`, guarded immutable nullable locals and eager
explicit fallbacks. A null lookup result is neither an empty REF nor an
implicit silent publication. This contract adds no temporal `get` overload,
invalid-child default rule, empty-REF constructor or node traversal through REF.
Unreadable keys and whole-map invalidation remain outside this extension.

See the [source examples](../../examples/map-key-connections.hgl),
[compiler cases](../../../compiler/cases_map_key_connections.md) and
[reference observations](../../../runtime/time_series.md#references).
