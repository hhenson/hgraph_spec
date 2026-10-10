# Growing-list publications

Admit `list<S>` (equivalently `list<S, unbounded>`) into the finite structural
publication profile when S is an admitted shape. Each input starts with length
zero and no publication. This is a temporal list of child endpoints; an
`atomic<list<S>>` remains a complete ordinary list snapshot.

## Delta construction and application

Use the ordinary structural delta constructor:

```hgl
delta<list<i64>>(items: [0: 10, 1: 20])
delta<list<i64>>(items: [1: 21, 2: 30])
delta<list<i64>>(items: [0: 11], remove: [1, 2])
```

`items` is a sparse list of constant nonnegative i64 indices and their child
deltas; `remove` is a list of constant nonnegative i64 indices. Either argument
may be omitted. Unknown/duplicate arguments or indices are checking errors.
As with other structural deltas, expressions are evaluated and independently
retained in written order. This adds no dynamic index construction API.

Canonical publication validation depends on the preceding input state. For
previous length L, nonempty `remove` must contain exactly one contiguous tail
`[c, L)` with `0 <= c < L`, without duplicates. Truncate to c before applying
`items`. In a removal delta, every modified index is below c: no position is
both removed and reintroduced. With no removals, updates below L preserve
length and newly appended indices must be exactly `[L, N)` for some N >= L,
without gaps. Every appended position carries a recursively admitted
child delta that makes the child valid, including an empty structural delta
under [EMPTY-1](empty-delta-validity.md). Membership without a valid child
publication is outside this profile.

Each included child follows its own delta application rule. Omitted existing
children retain their state. Equal scalar child publications still tick.
An explicit empty delta follows [EMPTY-1–2](empty-delta-validity.md): it
validates an invalid list without appending children, and is silent on a
valid list. A tail removal down to zero is a nonempty publication:
the root stays valid and modified that cycle, holds an empty list, and is
all-valid because it has no live invalid children. A later append starts at 0.
This is not whole-list invalidation.

These canonical deltas describe the net cycle under TS-12/TS-32. Intermediate
resize operations are not replayed as separate events. Runtime capture reports
all removed tail indices and modified child deltas, including newly joined
positions. Retained removed children remain a separate observation contract;
this publication delta does not capture their old values or endpoint identity.

## Eval, replay and recording

Use the same guarded generic pass-through, owning `delta<list<S>>`,
`TimedValue<list<S>>`, replay and record contracts. A pass-through applies the
net delta to its own output; it must not return a borrowed input endpoint or
substitute the held list. Harness equality compares removed indices as a set
and modified child deltas by index. Owning copies recursively retain these
ordinary deltas independently across subsequent growth/shrink and teardown.

Eval preflights the complete supplied trace, tracking length and recursively
validating children before any target starts. Gaps, non-tail removals,
out-of-range removals, removed/modified overlap and unsupported child events
use the existing input-profile error with parameter and zero-based position.
Exact typed wrappers retain empty/all-silent lifecycle and dense horizons.
Fixed-list size identity, invalidation and references keep their
existing boundaries.

See [examples](../../examples/growing-list-publications.hgl) and
[compiler cases](../../../compiler/cases_growing_list_publications.md).
