# Mutable ordinary value types

Status: proposed aggregate value-type extension, 2026-10-01.

An ordinary value type describes both its shape and whether its owner may
change its contents. `mut` selects the mutable form of an aggregate value
type. The unqualified form is immutable. This is distinct from rebinding a
local variable, from permission to write through a borrowed view, and from
the temporal shape of an endpoint.

This initial profile covers lists, tuples, structs, maps and sets whose
ordinary value types are already admitted. It does not settle mutable forms
of primitive scalars or opaque native atomics. It adds no collection mutation
API, nullable element type or first-class structural delta type.

## Type spelling, identity and construction

`mut` is a contextual prefix in a type position, not an expression operator.
It is the sole qualifier; `mutable` is not an alias. In ordinary value
positions write `mut list<T>`, `mut tuple<T0, T1>`, `mut S`, `mut map<K, V>`
or `mut set<T>`. A qualified generic type must resolve to an admitted
aggregate before use. Repeated `mut` qualifiers are rejected.

The qualifier is shallow. It changes the mutability of that aggregate, not
its children. `mut list<tuple<datetime, i64>>` has a mutable outer list and
immutable tuple entries. A child requiring a mutable type must say so in
its own type. This creates no second struct declaration or inheritance
family: a qualified struct retains its nominal name, generic arguments and
fields, with mutability included in its complete value-type identity.

Mutability is invariant in type matching, generic arguments and prepared
storage bindings. `list<i64>` and `mut list<i64>` are different exact value
types. Neither is implicitly converted to the other. A read-only view of a
mutable value is still a view of that mutable value type; reduced access
permission does not change its schema. A global-state binding includes the
qualifier just like its other type information.

An already admitted constructor may use its expected value type to construct
the mutable form directly. For example, given a struct `Box`,
`let box: mut Box = Box(amount: 1)` constructs an owned mutable Box. It does
not construct an immutable Box and convert it. With no mutable expected type,
the constructor selects its ordinary unqualified type. An existing value is
not a constructor: assigning an immutable Box to an expected `mut Box` is a
type error. Copying an owned value preserves its exact type, including the
qualifier and child qualifiers. A conversion or freezing operation would
need a separate contract; none is added here.

This expected-type rule does not complete ordinary typed empty list literals,
runtime tuple literals, list growth, or source operations for owning copies
of every aggregate. Those operations retain their separate admission rules.

## Bindings and authority

`let` and `var` describe the binding. `mut` describes the value type:

| Binding | Rebind the local? | Change its immediate aggregate contents? |
|---|---|---|
| `let x: S = ...` | No | No; S is immutable. |
| `var x: S = ...` | Yes, to another S | No; rebinding does not make S mutable. |
| `let x: mut S = ...` | No | Yes, with owner or authorized mutable-borrow access and an admitted operation. |
| `var x: mut S = ...` | Yes, to another mut S | Yes, under the same authority requirement. |

A mutable schema alone never supplies mutation authority. The owner has that
authority. A facility may explicitly lend it for a bounded lifetime, as
run-wide state does below. Ordinary value parameters, const configuration,
and temporal input reads provide read-only access, including when their
payload's exact type is mutable. No input consumer can change a producer's
value by qualifying its input as mutable. An immutable value may be replaced
in an owning slot; that replaces the value rather than mutating its contents.

Existing field-assignment syntax may replace an ordinary mutable struct's
field when the access is authorized and the assigned ordinary value matches
the field's type. A `let` binding does not prevent this content change.
Required fields cannot be unset. This extension does not add optional-field
clearing syntax or operations for growing containers. A field projection
preserves read-only access when its base is read-only; a child `mut`
qualifier cannot restore authority lost at that boundary. The qualifier on
a parent does not make an immutable child mutable. Conversely, a mutable
child type is not permission to write through a read-only parent view.

Shallow immutability restricts the aggregate's own fields, positions or
membership; it does not remove owner authority over an explicitly mutable
child. An owner-local `outer: Outer` whose field is `inner: mut Inner` may
write `outer.inner.amount` when amount belongs to that mutable child. It may
not replace `outer.inner`, because that field belongs to immutable Outer.
This distinction applies only to owner access: projecting through a read-only
input, parameter or borrowed parent never grants child mutation authority.

```hgl
struct Box {
    amount: i64
}

const fn replace_amount() -> i64 {
    let box: mut Box = Box(amount: 1)
    box.amount = 2
    return box.amount
}

test ordinary_mutable_struct {
    assert replace_amount() == 2
}
```

The example uses ordinary values only. Rebinding `box` is rejected. Changing
`amount` on an unqualified Box is rejected even if its binding uses `var`.
A parameter of type `mut Box` remains read-only under the ordinary parameter
contract; it cannot perform this assignment on its caller's object.

## Owning bindings and constructor retention

Initializing a new owning local from an ordinary value creates an independent
value copy. For example, after `let a: mut Box = Box(amount: 1)` and
`let b = a`, changing `b.amount` does not change `a.amount`. The inferred
type of b remains `mut Box`; a non-rebindable local can own mutable contents.
Rebinding an owning `var` likewise installs an independent value of its fixed
type. This is not an implicit shared mutable alias or a move that invalidates
the source. Physical copy elision or sharing is allowed only when those
observations remain unchanged.

An admitted constructor retains each ordinary value argument independently,
recursively including mutable children. A later source mutation cannot change
an already constructed parent. The exact child types and mutability qualifiers
are preserved. This is ordinary value construction, not a new clone operation.

A result explicitly specified as borrowed is different. Binding the borrowed
result of aggregate get preserves that borrow and its authority/lifetime;
a type annotation does not silently turn it into an owning copy. Another
local bound from that borrowed value remains subject to the alias rules
below. An explicit owning retention boundary must already be admitted by its
operation's contract. Read-only access to an input or ordinary parameter
never permits mutation of the original through that access; an independent
owned copy, once materialized, has its own owner authority.

## Temporal boundaries

`atomic` controls temporalization; `mut` controls the canonical value schema.
They are independent. In this initial profile, a mutable aggregate in a
temporal position requires an explicit atomic boundary. For an unqualified
aggregate V, `mut atomic<V>` normalizes to `atomic<mut V>`: the qualifier
belongs to the canonical payload. These are compositions of the same type
operators with one canonical result, not different mutability mechanisms:

```text
canonical(mut V) = mutable ordinary aggregate V

mut atomic<V> = atomic<mut V>

temporalize(atomic<mut V>)
    = one atomic endpoint carrying canonical(mut V)
```

For example, `mut atomic<list<i64>>` is one endpoint whose complete scalar
payload has type `mut list<i64>`. It is not a structural list of temporal
children and does not make the endpoint's connection mutable. Input access
is read-only; output changes still require the owning node's ordinary
publication/output rules. A mutable payload does not publish merely because
an unrelated local or store value changes.

Unwrapped `mut list<...>`, `mut tuple<...>`, `mut S`, `mut map<...>` and
`mut set<...>` are rejected in temporal positions in this profile. Do not
erase the qualifier or silently insert an atomic boundary. Both placements
of the single qualifier above resolve to the same exact payload and temporal
type. Qualifying the same aggregate twice, as in `mut atomic<mut V>`, is
rejected. A child qualifier inside V remains part of that child's ordinary
schema. Generic substitution preserves those qualifiers before checking
whether a temporal aggregate has the required atomic boundary.

Ordinary value contexts need no atomic boundary: a stored recording can have
value type `mut list<tuple<datetime, i64>>`, while ordinary const replay data
can have type `list<tuple<datetime, i64>>`. `const data: atomic<V>` and
`const data: mut atomic<V>` (equivalently `atomic<mut V>`) are invalid:
const parameters already bypass
temporalization. `const data: mut V` describes mutable-schema data supplied
as read-only configuration, not permission to mutate that configuration.

## Views, owning copies and retained data

A read-only value view follows VAL-16: it is a stable observation, not a
mutable alias exposed to a consumer. A borrowed owner view is different: it
is explicitly authorized live access to mutable contents, bounded by its
provider's lifetime. Its own admitted mutations are visible through that
access. The read-snapshot stability rule does not promise that a mutable
owner view is frozen.

Owning copies follow VAL-17 recursively. A retained copy sees no later changes
to the original or its children, regardless of whether its schema is mutable.
A new owner may subsequently mutate its own mutable copy. Physical sharing
is permitted only when these independence guarantees hold. Retaining a view
handle is not an owning copy. Ordinary source assignment does not implicitly
turn a borrowed owner view into an independent owned aggregate.

Where an existing operation explicitly retains an ordinary value into owned
storage, it retains a copy of the value, not the borrow. The operation's type
and ownership contract must admit that argument. A generic source copy,
freeze or clone function is not introduced by this extension. Insertion into
a future mutable recording must retain independent entries; it must not copy
the entire recording merely to acquire mutable access to it.

## Typed global-state borrowing

The prepared entry rules of [ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md)
remain in force. Keys are const, exact types include mutability, bindings
are prepared before start, and runtime get checks presence without key lookup
or type dispatch. Mutable aggregate access introduces no new getter or
recording-specific entry type.

An aggregate `get(global_state, key)` supplies a hook-bounded borrowed value.
A mutable aggregate entry lends mutable owner access; an immutable aggregate
entry supplies read-only access. Bind either with an explicitly typed `let`
local. The result has its ordinary value type; the checker also tracks its
borrow provenance and lexical lifetime. No source borrow type is added.
Primitive scalar reads keep their existing owned-copy behavior. An immutable aggregate
containing mutable children does not expose those children as writable through
this read-only access. No borrowed view survives into another hook. An
independently retained snapshot must copy all nested mutable data; a
read-only outer wrapper alone does not provide that independence.

For this profile, the borrow lasts to the end of the lexical block containing
its local and never beyond the hook. A borrowed aggregate cannot initialize
`var`, escape via a closure or ordinary helper call, or be retained as a
borrow in state, cache, another entry or an output. An operation explicitly
specified to take an owning copy may consume its ordinary value; the borrowed
access itself still ends at its lexical boundary. Construction and mutation
operations not yet admitted by their value contracts remain unavailable.

A mutable borrow is exclusive. While it is live, a second get of that entry
or any set replacing its value is rejected. A read-only borrow may coexist
with other read-only borrows; replacement or a conflicting mutable borrow
is rejected while any of them is live. Creating another local alias of a
mutable borrow is rejected; it is not an implicit owning copy. Read-only
aliases inherit the same borrow and do not extend it beyond the originating
block. All derived borrowed projections end no later than their parent.
An aggregate projection cannot be bound as an additional live view when it
would conflict with the originating mutable access. Materializing an
independent owned copy through an admitted copying operation avoids that
alias; merely annotating a local with an immutable type does not.

Known conflicts are checking errors. If const key parameters become equal
only when a graph is configured, required distinctness is validated before
start and a conflict fails construction. Checking tracks existing control
flow: disjoint branch lifetimes do not overlap. There is no per-evaluation
borrow registry, key lookup or dynamic type test. A fresh hook may borrow
the entry again after the previous hook's borrow ends.

For an entry already seeded with a `mut Box`, an ordinary sink can change its
owned field through the borrowed access without replacing or copying the
complete stored value:

```hgl
fn increment_entry(value: i64, const key: str) {
    inject global_state
    when {
        let box: mut Box = get(global_state, key)
        box.amount += 1
    }
}
```

The entry's exact type is fixed before start. Its absence still makes get
fail. The mutation is authorized by the borrowed owner access, not by the
fact that the local uses `let` or that the node has an i64 input.

For example, the two accesses below conflict if first and second are the same
configured key. If they are distinct, both typed entries may be borrowed:

```hgl
fn inspect_entries(value: i64, const first: str, const second: str) {
    inject global_state
    when {
        let a: mut list<i64> = get(global_state, first)
        let b: mut list<i64> = get(global_state, second)
    }
}
```

The bindings illustrate access checking only; this extension supplies no
`push` operation for those lists. Ordinary list construction, read/index,
length and mutation contracts follow separately. Non-scalar cache
construction, structural delta storage and complete replay/record bodies
also remain separate. No historical recording is reclassified as cache.
