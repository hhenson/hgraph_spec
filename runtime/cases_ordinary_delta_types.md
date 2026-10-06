# Ordinary publication-delta type cases

These expected observations exercise
[ordinary delta types](../language/docs/design/ordinary-delta-types.md).
Type formation and retention do not imply admission of a publication event.

## DELTA-SYNTAX — type marker and constructor

In an ordinary value context, the annotation names the delta type and the
parenthesized form constructs a value of that type:

```hgl
let update: delta<map<i64, i64>> = delta<map<i64, i64>>(upsert: [1: 10])
let scalar: delta<i64> = 7
```

The same marker is admitted in ordinary fields, generic arguments and
value-function parameter/result types. These examples add no scalar
constructor form. `delta_value(endpoint)` remains the guarded temporal
accessor; the shared type/constructor spelling does not make it an alias
for that accessor.

## DELTA-TYPE — canonical identity

`delta<i64>` is i64 and `delta<str>` is str. The same reduction holds
for all admitted scalar types. No runtime wrapper is observed.

`delta<list<i64, 2>>` and `delta<list<i64, 3>>` are different types,
even for data containing only index zero. Two nominal structs with identical
fields have different delta types. Fully applied nominal argument lists,
tuple positions, child types and container kinds remain part of identity.
An ordinary held map cannot initialize its delta type and a delta cannot
initialize that held map type. No sparse-layout compatibility substitutes
for exact shape identity.

Forming `delta<T>` for an excluded concrete temporal shape fails checking.
Using a structural delta type as a new temporal endpoint payload is not
admitted here. A zero-size fixed shape may have a formed delta type without
acquiring any admitted nonempty publication.

## DELTA-INFER — symbolic formation and matching

A typed empty `list<TimedValue<map<i64, i64>>>` passed to generic
replay fixes T to `map<i64, i64>`, despite having no payload from which to
infer anything. Scalar timed i64 data fixes T to i64. Matching actual
`delta<Quote<i64>>` binds that complete nominal specialization, not an
anonymous set of matching fields. Repeated incompatible bindings fail.

A generic body may retain symbolic `delta<T>` and its formation obligation.
Instantiation must resolve T to an admitted type. An untyped empty list
does not determine T; no key spelling or runtime value supplies missing
type information. Typed global entry bindings use the resulting exact type
before start, without runtime type-name lookup or type tests.

## DELTA-OWN — copy, store and replacement

Timed entries below use `TimedValue<T>` for the original temporal shape T;
their value field has type `delta<T>`. Thus a timed map publication uses
`TimedValue<map<i64, i64>>`, not a TimedValue parameterized by a structural
delta type. `TimedValue<i64>` still contains an ordinary i64 payload.

A complete held map cannot initialize that timed map's value field. A delta
of a different originating shape also fails checking. TimedValue applications
with different fixed sizes or nominal shape arguments remain distinct types.
Passing a structural delta type as T fails its `delta<T>` formation requirement;
the former payload-type parameter convention is not a compatibility path.

For each of these ordinary delta shapes, retain a first value A into a timed
entry, a list and an exact typed global entry. Obtain an independent owned
recording result under an admitted retention boundary. Replacing the source
owner with B and destroying it does not change A in any retained result.

| Shape | A | B |
|---|---|---|
| i64 set | add 1 and 2 | remove 1 |
| fixed i64 list size 3 | index 0 → 10, index 2 → 30 | index 0 → 11 |
| named bundle with bid/ask i64 fields | bid → 10, ask → 20 | bid → 11 |
| i64-key i64 map | keys 1 → 10, 2 → 20 | key 1 → 11 |
| nested i64-key map | key 7 → keys 1 → 10, 2 → 20 | key 7 → key 1 → 11 |
| fixed list of i64-key maps | index 0 → keys 1 → 10, 2 → 20 | index 0 → key 1 → 11 |

These are ownership observations, not ordinary source equality operations.
HGL conformance may observe the retained values through already admitted
publication/harness comparison using canonical nonempty traces.

A read-only or writable typed global-entry borrow retains its lexical
lifetime rules. A typed immutable local holding a structural delta_value
observation cannot escape merely because it has an ordinary type annotation.
Constructor/list/global retaining arguments may copy its data independently.
Writable authority does not permit a new delta-field mutation operation.

## DELTA-CONSTRUCT — checking, expression order and failure

Validate all names, duplicates, constant keys/member lists, overlap, indices
and child types before executing payload expressions. An invalid later
entry prevents even an earlier valid payload expression from running.

For a well-typed fixed-list constructor with written entries 2 then 0,
payload effects occur for index 2 before index 0, regardless of index order.
Each value is retained before the next expression starts. A nested delta
constructor completes its own sequence before the outer sequence continues.
For a named bundle written ask then bid, effects occur ask then bid even if
the schema declares bid first. The final sparse data still associates values
with their written field/index/key identities.

If the first payload expression or its retention fails, the second is not
evaluated. If a later expression fails, earlier effects stand. Neither case
produces a partial delta. These rules concern expression effects; they do
not apply sequential membership changes to an endpoint during construction.

## DELTA-DATA — storage is not a publication

An ordinary empty map delta may be retained in a list. The list has one
element, not zero; this creates no tick. Empty data is neither null nor the
harness `_`. This case does not decide whether applying that data as an
empty event is admitted or what it would do.

A well-shaped removal can be stored without knowing a target's membership.
Applying it still needs the existing state-dependent canonical-removal
precondition. Storing a value does not bypass fresh eval trace validation.
Publication/invalidation information not carried by the bounded delta
profile cannot be reconstructed from omitted children or default values.

## DELTA-OPERATIONS — bounded ordinary API

Admit exact-type ordinary passing, copying, value-function returns, retaining
construction/list insertion/global storage, owning replacement and matching
runtime delta publication. Structural-delta field/index inspection, iteration,
mutation, ordinary equality, ordering, hashing and conversions remain
unadmitted. Existing harness sparse comparison and ordinary scalar operations
on scalar reductions retain their separate rules.
