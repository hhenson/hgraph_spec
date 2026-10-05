# Atomic set and map publications

Extend the finite atomic payload grammar with ordinary `set<K>` and
`map<K, V>`. K uses the [scalar-key contract](scalar-collection-keys.md),
including its non-NaN f64 boundary; V recursively uses the admitted finite
ordinary payload grammar, including these sets/maps. Existing struct-field rules remain, including
[optional atomic fields](optional-atomic-publications.md). Recursive nominal
definitions and references are not added.

## Ordinary construction

Construct complete ordinary values with these exact typed forms:

```hgl
set<str>(items: ["alpha", "beta"])
map<str, list<i64>>(items: ["alpha": [1, 2], "beta": []])
set<str>(items: [])
map<str, i64>(items: [])
```

`items` is the sole required named argument. This is ordinary construction,
not a temporal mutation or delta constructor. The expected types are K for
set items and map keys, and V for map values. K and V must be fixed by the
explicit type application; an empty constructor does not infer them.
Map entries use `expression : expression` within this constructor's `items`
list. Outside this form, existing list, sparse-delta and timed-entry grammars
are unchanged; no untyped map literal is introduced.

Member, key and value expressions may be ordinary expressions in the current
phase. They need not be constants. This does not change the constant-entry
restriction of sparse temporal delta constructors. Evaluate in written order:
each set member; for maps each key then its value, before the next pair.
Independently retain each evaluated ordinary value before evaluating the next
expression. No returned container or retained child aliases its source.
Provider-dependent cold values follow existing context and validation rules.

Reject duplicate members/keys by K equality, including signed-zero duplicates.
Known duplicates are checking errors; otherwise the constructor fails in its
executing phase as soon as the duplicate member/key is encountered, before a
duplicate map key's value expression is evaluated. Type mismatch, failed
construction or unsupported key values cannot expose a partially constructed
container or publish a partial result. Already evaluated expression side
effects are not rolled back; later expressions are not evaluated.
This adds construction and owning replacement, not new element mutation APIs.

## Complete publication and capture

For `atomic<set<K>>` and `atomic<map<K, V>>`, the delta is the complete ordinary
set/map. Publication replaces the held value; it does not apply membership
operations, merge keys or recursively patch map values. Omitting an old key
removes it from the complete snapshot. An empty set/map is present data,
including when nested inside another finite atomic value or a sparse outer
publication. Equal complete snapshots still publish; `_` alone is silence.

Use the existing generic pass-through, guarded `delta_value`, `TimedValue`,
eval, replay and record contracts. Ordinary equality and harness comparison
ignore set/member and map/entry order while preserving exact K and recursive V
identity. Independently retain complete snapshots at every existing owning
boundary. Source mutation, graph teardown, another eval or a mutation through
one writable owned copy cannot change another capture, including nested map
values. Read-only native storage does not grant source mutation permission.

The composite payload alone does not infer its temporal shape: an exact atomic
wrapper or existing type-bearing context still fixes that boundary. Empty
structural publications and invalidation remain separate from these present
complete values.

See [examples](../../examples/atomic-set-map-publications.hgl) and
[compiler cases](../../../compiler/cases_atomic_set_map_publications.md).
