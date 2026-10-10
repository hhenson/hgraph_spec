# Contextual collection publication deltas

Status: source extension, 2026-09-30. This defines ordinary collection
publication deltas, extended by the accepted
[empty sparse application rule](empty-delta-validity.md).

## Shape-derived delta relationship

Extend the [delta-value contract](delta-value-metadata.md) to the following
finite temporal shapes T:

- the thirteen built-in scalar leaves `bool`, `i64`, `f64`, `str`, `bytes`, `date`, `time`,
  `datetime`, `duration`, `civil_datetime`, `timezone`, `zoned_datetime`, and `zoned_time`,
  with bytes under [BYTE-1–6](bytes-values.md);
- `any` leaves under [ANY-1–5](any-values.md);
- native atomic leaves under [NVAL-1–5](native-atomic-values.md);
- declared enum leaves under the [enum publication contract](enum-publications.md);
- `set<K>` for the [admitted scalar keys](scalar-collection-keys.md);
- fixed `list<S, N>`, with a constant nonnegative size N;
- growing `list<S>` under the [net growing-list contract](growing-list-publications.md);
- positional `tuple<S0, ...>` and fully applied concrete nominal structs;
- `map<K, S>` for those same scalar keys;
- exact rolling shapes under the [arrival delta contract](rolling-publications.md).

Every child S or struct field must recursively have one of these shapes.
Recursive nominal definitions, references, signals,
other scalar types are outside this profile. The
[finite atomic extension](atomic-delta-publications.md) additionally admits
atomic children with complete ordinary payloads, including empty lists;
outer sparse publications follow their own application rules.

`delta<T>` names T's publication-delta shape. The
[ordinary delta type extension](ordinary-delta-types.md) admits this spelling
as an ordinary source type expression. For a scalar it is
that scalar. For a set it is disjoint added/removed members. For a fixed
structure it is sparse child deltas by field or position. For a map it is
removed keys and sparse child deltas by key. It is not T's complete held
value. Omitted children have no delta entry; they are not read as scalar
values.

`delta_value(value)` reads that contextual delta when the particular
endpoint is valid and modified. Apply the existing guard rules after
supplying implicit `when` selectors. Top-level `valid` suffices for a
partially valid bundle; `all_valid` is not required. An any-input-modified
selector alone does not prove a particular input modified.

The generic compute remains:

```hgl
fn pass_through<T>(value: T) -> T {
    when { return delta_value(value) }
}
```

A contextual delta may appear in a matching runtime return or own-output
assignment, a contextual immutable `let`, a nested delta constructor, or an
explicitly admitted harness/capability delta position. Ordinary type
annotations, owned construction and storage use the separate delta type
extension; state/cache admission limits remain unchanged. A borrowed input
delta remains evaluation-local, and applying or independently retaining it
must not retain a borrow into that input. Scalar specializations retain
ordinary scalar rules.

## Constructors and sparse entries

The existing `delta<Struct>(field: expression, ...)` constructor keeps its
omission and no-default behavior. Add these shape-specific forms:

| Shape | Constructor |
|---|---|
| Set | `delta<set<i64>>(added: [1, 2], removed: [3])` |
| Fixed list | `delta<list<i64, 3>>(items: [0: 10, 2: 30])` |
| Positional bundle | `delta<tuple<i64, str>>(items: [1: "ask"])` |
| Named bundle | `delta<Quote>(bid: 1)` |
| Map | `delta<map<i64, i64>>(upsert: [7: 10], remove: [9])` |
| Recursive child | `delta<map<i64, Quote>>(upsert: [7: delta<Quote>(bid: 1)])` |

All arguments are named. Set arguments are `added` and `removed`; fixed
list/tuple has `items`; map has `upsert` and `remove`. Omitted arguments
contain no entries. Nominal structs use their actual field names.

The delta argument grammar extends the ordinary named-argument grammar:

```ebnf
delta_arguments = [ delta_argument, { ",", delta_argument }, [ "," ] ];
delta_argument  = member_name, ":", ( expression | sparse_entries );
sparse_entries  = "[", [ sparse_entry, { ",", sparse_entry }, [ "," ] ], "]";
sparse_entry    = const_expression, ":", expression;
```

A sparse entry list is admitted only for fixed list/tuple `items` or map
`upsert`. It adds no general map-literal syntax and does not reinterpret
timed harness entries. Fixed indices are constant in-range i64 values;
map keys are constants of their exact admitted scalar K under the
[scalar-key extension](scalar-collection-keys.md). Set member
lists and map `remove` lists contain constant values of their exact element
or key type. This bounded literal grammar does not add dynamic-key or
runtime membership-list construction.

The context of each entry is its child's delta shape: a scalar child takes
an ordinary scalar expression; a structural child takes a matching contextual
delta expression. Unknown arguments/fields, duplicate arguments/fields,
duplicate member/index/key entries, out-of-range indices, set added/removed
overlap, and map upsert/remove overlap are checking errors. Entry order is
not an ordered sequence of mutations. Payload expressions and their retention
follow the [written-order construction rule](ordinary-delta-types.md#construction-order-and-failure).
Empty argument/entry data may be constructed and applied under
[EMPTY-1–4](empty-delta-validity.md). Existing optional-field clearing is
not redefined: explicit `null` and invalidation are outside this publication
profile, and omission continues to mean no change.

## Ordinary publication application

A sparse update contains set membership changes, map removals or child
deltas, each applied to its own target. Explicit empty deltas follow
[EMPTY-1–2](empty-delta-validity.md), including at nested children. A child
scalar publication is present even if its value equals its previous value.

`return d`, where d has exact derived type `delta<T>`, applies d to the
runtime node's own T output and terminates evaluation. `out = d` applies the
same update and continues. These operations never return an input port or
replace the node with identity wiring. Complete-value returns remain a
separate operation.

- Fixed structures update only the named children. Their other values and
  validity remain unchanged; omitted children do not tick.
- Set additions/removals change the indicated memberships.
- Map removals remove keys. Upsert of a present key recursively applies its
  child delta without replacing untouched descendants. A new key receives a
  child delta that makes that child valid; creating an invalid child alone
  is outside this profile.
- A valid child publication makes its owned ancestors modified. Equal scalar
  child publications still tick. Multiple writes in one evaluation retain
  the existing accumulation and last-write rules.

These rules specify canonical publication transitions: set additions refer
to absent members, removals to present members, and map removals to present
keys. Invalid nonempty membership instructions are rejected; they are not
empty updates. Eval reports them through its existing input-profile error.
In particular, constructor argument names do not desugar to the named output
mutation functions. The existing strict `insert`/`update`/`remove` and tolerant
`upsert`/`discard` APIs keep their separate preconditions and behavior.

For example, after a map holds key 7 with inner keys 1 and 2, applying
`delta<map<i64, map<i64, i64>>>(upsert: [7: delta<map<i64, i64>>(upsert: [1: 11])])`
publishes only the path through 7 and 1. Inner key 2 retains its held value
and is absent from the new publication delta.

## Deliberate boundaries

An actual empty set delta can accompany a tick after same-cycle cancellation
([collections within a cycle](../../../runtime/time_series.md#collections-within-a-cycle)).
Applying that payload follows [EMPTY-3](empty-delta-validity.md): an invalid
target ticks empty and becomes valid; a valid target does not tick. No-tick
`_` is not an encoding of empty data. A nonempty removal that leaves a
collection empty remains a publication.

Publication deltas are not complete endpoint-state changes. Whole/child
invalidation, creation of invalid map children, independent membership
signals, unbinding and REF designation need separate contracts; do not encode
them using omitted fields, `_`, null or default scalar values. The runtime's
validity/membership rules remain applicable beyond this bounded publication
profile.
