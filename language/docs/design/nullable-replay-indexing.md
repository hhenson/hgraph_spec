# Nullable indexed replay slots

Status: replay-access contract correction, 2026-09-30.

A replay source reads its configured sequence with ordinary indexing:
`replay_input[index]`. The index is an i64, is zero-based, and counts every
slot, including absent slots. `len(replay_input)` counts those same slots.
Neither operation uses a timestamp or advances a cursor.

## Presence, bounds and ownership

An in-range read returns the supplied owned delta when the slot is present,
and the existing language absence literal `null` when it is absent. It never
returns a previous held value or a default scalar. In the scalar profile the
present payload is T; in the collection profile it is contextual
`delta_of(T)`. The latter remains specification notation, not a source type.

| Input sequence | Expression | Result |
|---|---|---|
| `[10, _, 12]` | `len(replay_input)` | `3` |
| `[10, _, 12]` | `replay_input[0]` | Present `10`. |
| `[10, _, 12]` | `replay_input[1]` | `null`. |
| `[10, _, 12]` | `replay_input[2]` | Present `12`, not the arithmetic difference `2`. |
| Any sequence of length n | Index less than 0 or at least n | Bounds failure. |

Indexing is evaluation-only; `len(replay_input)` is admitted in start and
evaluation. Neither is admitted in stop. Validate `0 <= index < len(replay_input)`
before accessing a slot. A bounds failure uses the existing translated node
error contract and message prefix `replay_input: index out of range`, leaving
input and capture unchanged. An absent in-range slot is a successful read,
not a bounds or missing-publication error.

A successful present read owns its payload recursively. Repeated reads of
one immutable slot return the same presence and payload without sharing
mutable borrowed views. Reading never publishes, schedules, consumes a slot,
changes a cursor, or materializes a dense result. `delta_value(endpoint)`
continues to read a live endpoint's current publication; it is distinct from
configured replay-slot access.

There are no `delta_at` or `has_tick` replay operations or compatibility
aliases. Presence is tested on the indexed result. The capability itself
retains its existing node/run binding and non-escape rules.

## Contextual nullable result and flow refinement

The index result has a contextual nullable payload: either absence or the
exact admitted delta. This extension introduces no source `Option`, nullable
type constructor, annotation or general union type. The result may initialize
an inferred immutable `let` local within the current evaluation. It may be
compared with `null` using `==` or `!=`. Such a comparison is a presence test,
not scalar equality, structural delta equality or a temporal operator call.

Before presence is established, no payload use is admitted: no arithmetic,
ordering, field/index projection, runtime output assignment, value return,
constructor argument, capture append, or ordinary helper/operator argument.
There is no implicit unwrapping or substitution. A nullable bool is not a
condition and has no truthiness conversion. False, zero and empty text are
present scalar payloads; a present structural payload is not absent merely
because its delta contains no entries. Whether applying an empty structural
publication is admitted remains its separate contract.

For an immutable local `item` initialized by a replay read, direct comparisons
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
let item = replay_input[current]
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

## Replay and capture

The source preserves its existing cursor and scheduling policy. The relevant
evaluation body is:

```hgl
when {
    let current = index
    index += 1
    if index < len(replay_input) {
        schedule_at(alarm, clock.next_cycle_evaluation_time)
    }
    let item = replay_input[current]
    if item != null {
        return item
    }
}
```

An absent read follows the path without return/output mutation and therefore
publishes nothing. This does not define `return null` as a general no-output
operation. `null` used to clear an optional struct field retains that field's
separate meaning; it is not replay silence. Harness `_` remains the external
no-publication slot spelling, and this correction does not add `null` as a
harness-slot alias.

`begin(capture)` and `append(capture, time, delta)` keep their existing phases,
ownership, timestamp checks and errors. Append requires a non-null delta;
it never accepts absence as a request to skip, pad or erase a recording.
The normal record body still calls
`append(capture, last_modified(ts), delta_value(ts))`.

See [ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md) for the full
scalar source body and [collection eval](eval-collection-deltas.md) for
recursive shape admission. Capability calls use
[receiver-first syntax](capability-function-syntax.md).
