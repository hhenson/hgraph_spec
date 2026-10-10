# Nullable sequence indexing and presence refinement

Status: contextual presence contract, 2026-09-30.

For an ordinary sequence whose element contract admits present and absent
slots, use `result[index]`. The index is an i64, is zero-based, and counts
every slot. `len(result)` counts the same slots. These are value operations,
not injectable operations. Neither advances a cursor or uses a timestamp.

The rules below preserve the presence behavior needed by eval. They do not
supply a general source sequence type, its construction, a nullable type
annotation or a nullable structural delta container. Nullable sequence
formation remains open in [ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md#unresolved-source-contracts).
The [ordinary delta type extension](ordinary-delta-types.md) separately
supplies non-nullable owned publication data.
In the examples, `result` denotes an already admitted ordinary sequence;
the bracket notation in the table describes its slots using harness notation.
It is not a new ordinary value constructor.

## Presence, bounds and ownership

An in-range read yields its present payload or the existing absence literal
`null`. It never returns a held value or default scalar. A scalar payload has
its ordinary scalar type; a structural publication payload retains the
contextual `delta<T>` relationship where separately admitted.

| Sequence slots | Expression | Result |
|---|---|---|
| `[10, _, 12]` | `len(result)` | `3` |
| `[10, _, 12]` | `result[0]` | Present `10`. |
| `[10, _, 12]` | `result[1]` | `null`. |
| `[10, _, 12]` | `result[2]` | Present `12`, not the arithmetic difference `2`. |
| Any sequence of length n | Index less than 0 or at least n | Bounds failure. |

Require `0 <= index < len(result)` before accessing a slot. Out-of-bounds
access fails under the applicable value-operation error contract; inside a
node hook it follows the translated node error contract. An absent in-range
slot is a successful read. A missing global-state key is a separate error,
not an absent sequence element.

Indexing follows the sequence value's lifetime and ownership contract, not a
replay-specific phase restriction. A retained capture must own its data
independently; indexing alone does not authorize a borrowed aggregate view
to escape. Reading does not publish, schedule, consume a slot, or materialize
a dense result. `delta_value(endpoint)` instead reads a live endpoint's
current publication.

## Contextual nullable result and flow refinement

The index result has a contextual nullable payload: either absence or the
exact admitted element payload. This extension introduces no source `Option`, nullable
type constructor, annotation or general union type. The result may initialize
an inferred immutable `let` local within the current hook. It may be
compared with `null` using `==` or `!=`. Such a comparison is a presence test,
not scalar equality, structural delta equality or a temporal operator call.

Before presence is established, no payload use is admitted: no arithmetic,
ordering, field/index projection, runtime output assignment, value return,
constructor argument, collection insertion, or ordinary helper/operator argument.
There is no implicit unwrapping or substitution. A nullable bool is not a
condition and has no truthiness conversion. False, zero and empty text are
present scalar payloads; a present structural payload is not absent merely
because its delta contains no entries. Applying it follows the
[empty sparse application contract](empty-delta-validity.md).

For an immutable local `item` initialized by an indexed read, direct comparisons
establish these facts:

| Condition | True branch | False branch |
|---|---|---|
| `item != null` or `null != item` | Present: payload usable. | Absent: payload unusable. |
| `item == null` or `null == item` | Absent: payload unusable. | Present: payload usable. |

Parentheses preserve these facts. Logical negation swaps them. Existing
short-circuit rules propagate the left operand's true facts into the right
operand of `&&`, and its false facts into the right operand of `||`.
Consequently `item != null && predicate(item)` and
`item == null || predicate(item)` may use the payload in their right operand.
Facts in a compound condition's branches follow those evaluation paths;
at a merge, retain only facts true on every reachable incoming path. A
terminating branch contributes no continuing path. Thus this is admitted:

```hgl
let item = result[index]
if item == null { return }
return item
```

A fact belongs to the particular immutable local binding. Shadowing introduces
a different binding. No fact is inferred from an arbitrary helper predicate,
a saved comparison bool, scalar truthiness or equality with a non-null
scalar. Comparing one indexed expression does not refine a later read;
bind the result once and test that binding.

`let alias = item` before item is refined creates another nullable local
requiring its own presence guard; guarding item does not guard alias. Copying
a proven-present item instead initializes an ordinary scalar or non-null
contextual delta local, subject to the existing rules for that payload.
A nullable local may not enter `var`, state, cache, a closure, a collection,
a struct field, an ordinary parameter/result or any other escaping storage.
Refinement does not relax structural-delta escape restrictions.

These are checking rules. A rejected nullable use cannot be lowered to an
absent publication, a default scalar or an unchecked payload read.

## Publication and other uses of null

Given an admitted sequence and a matching output context, this fragment
illustrates guarded use; it is not a complete replay implementation:

```hgl
let item = result[index]
if item != null {
    return item
}
```

An absent read follows the path without return/output mutation and therefore
publishes nothing. This does not define `return null` as a general no-output
operation. `null` used to clear an optional struct field retains that field's
separate meaning. Harness `_` remains the external no-publication slot
spelling; this contract does not add `null` as a harness-slot alias.

Presence refinement makes a payload usable only in a context already
admitting its type and ownership. Ordinary delta storage uses its own
type/retention contract; presence refinement does not provide an owning
copy or permit a nullable local to escape.
