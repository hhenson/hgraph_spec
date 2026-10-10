# Ordinary value mutability

Status: proposed binding and access extension, 2026-10-01.

`let` gives read-only access; `var` gives writable access. Global-state entries
are mutable storage. These are permissions over ordinary values, not separate
value types or new type qualifiers. An ordinary value V has the same canonical
type in either binding. Temporalization, generic type matching and the
`atomic<V>` boundary remain unchanged. Inputs remain read-only.

This contract defines ownership, assignment and lexical aggregate borrowing.
It adds no list construction, growth or other collection API, nullable element
type, or first-class structural delta type. The separate
[delta type extension](ordinary-delta-types.md) admits ordinary publication
deltas under these same ownership rules. Each operation still needs its
ordinary value contract. The separate
[ordinary-list extension](ordinary-list-values.md) defines typed empty
construction, length, indexed reads and retained end growth under these rules.
The [ordinary set/map construction extension](atomic-set-map-publications.md)
adds complete typed constructors with independent retained children.

[Contextual local bindings](contextual-local-bindings.md) fixes each local’s
ordinary-value or connection category as well as its canonical type. Mutability
does not permit changing either.

## Bindings and projections

| Ordinary access | Replace the value? | Mutate its contents? |
|---|---|---|
| Owning `let x: V = ...` | No | No |
| Owning `var x: V = ...` | Yes | Yes, using admitted operations |
| Parameter, const configuration, input or `for` binding | No | No |
| Aggregate get bound to typed `let` | No | No; lexical read-only borrow |
| Aggregate get bound to typed `var` | Yes, in the borrowed entry | Yes; exclusive lexical borrow |

Read-only access is recursive. A `let outer: Outer` cannot assign its inner
field or mutate that child's contents. No projection restores write authority
lost at a read-only boundary. An owning `var outer: Outer` may do both when
the field's ordinary type admits the operation. Complete construction must
populate required fields; ordinary mutation cannot clear them. Retention from an
admitted partial structural observation instead preserves typed absence under
[required-read rules](unset-required-reads.md); it introduces no field-clearing
syntax or mutation permission.

For example, ordinary field assignment uses the existing syntax:

```hgl
struct Box { amount: i64 }

const fn replace_amount() -> i64 {
    var box: Box = Box(amount: 1)
    box.amount = 2
    box = Box(amount: 3)
    return box.amount
}
```

Changing the declaration to `let` makes both assignments checking errors.
The type remains Box. Ordinary parameters provide read-only access even when
the caller supplied an owning `var`.

These rules concern ordinary values. Composition-level `var` wire rebinding
continues to change a local connection handle, not a producer's value. Neither
wire rebinding nor ordinary content mutation authorizes writes to temporal
inputs. Output publication and persistent state keep their existing contracts.

## Owning bindings and retained values

Initializing a new owning local from an ordinary value creates an independent
copy. After `let a: Box = Box(amount: 1)` and `var b = a`, assigning
`b.amount = 2` leaves a unchanged. Conversely, `let b = a` makes that new
owning copy read-only through b. Rebinding an owning `var` installs an
independent value of its fixed type; it neither moves from the source nor
creates a shared mutable alias.

An admitted constructor retains ordinary value arguments independently,
recursively. For `struct Envelope { inner: Box }`, constructing
`Envelope(inner: source)` then changing source cannot change the retained
child. Copies preserve canonical types, not the access permissions of the
source binding. Physical sharing or copy elision is allowed only when these
observations remain unchanged (VAL-17). Ordinary struct constructors
[evaluate and retain each supplied argument in written order](struct-constructor-order.md)
before proceeding to the next argument.

A result explicitly specified as borrowed is different: binding aggregate
get preserves its borrow provenance and lifetime. An ordinary initializer
cannot silently turn such a borrowed result or projection into an owning
copy. An operation explicitly specified to retain an owning value may copy
it; admitted constructor arguments and global-state set are such retention
boundaries. Assignment into an already owning `var` or its writable field is
also an owning retention boundary: a borrowed right-hand value is copied
independently before replacement. It does not change the destination into a
borrow. For example, assigning a borrowed Box to an already owning Box local
is permitted, whereas initializing another local from the borrow must obey
the alias restrictions below. No new clone function or source borrow type is
introduced here.

## Views and storage

A consumer's read-only observation follows VAL-16: it is stable within its
admitted lifetime, not a writable alias. Authorized owner access is live:
it observes its admitted changes and is not that consumer snapshot. A
read-only lexical global-entry borrow is protected from conflicting writes
by the rules below. It cannot be kept as a snapshot merely by retaining its
view handle.

[Required reads of unset retained observations](unset-required-reads.md) fail
when an operation needs an absent payload; retention itself preserves absence.

Retained owning copies follow VAL-17 recursively. Later changes to an
original value or any child cannot appear through its retained copy. A new
owning `var` may change its own copy. This independence applies to recording
entries as well as other retained values; retaining a captured value must
not retain a live view of its source.

Physical mutable and read-only representations may differ without defining
different HGL types. A provider promising writable owner access must supply
storage supporting that access even when the original value used read-only
storage. In particular, global entries support `var`-authorized mutation
without copying the entire entry merely to obtain a borrow. This promises
no allocation-free payload construction, assignment or insertion.

## Typed global-state borrowing

[ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md) still determines
const keys, concrete value types, preparation before start and required-read
presence errors. Binding/access mode is not part of entry type identity.
Aggregate entries are mutable storage; get does not infer type from node role
or add per-hook key lookup or dynamic type dispatch.

For aggregate V, bind read-only access with:

```hgl
let observed: V = get(global_state, key)
```

Alternatively, bind exclusive writable access with:

```hgl
var editable: V = get(global_state, key)
```

Each form requires the expected concrete ordinary type. The first borrows
read-only access. The second borrows exclusive writable access to the existing
entry, without making an owning local copy. Both are available in start,
evaluation and stop hooks. An uninitialized entry still makes get fail.
Primitive reads of `bool`, `i64`, `f64`, `str`, `bytes`, `date`, `time`, `datetime`,
`duration`, `civil_datetime`, `timezone`, `zoned_datetime` and `zoned_time` retain their
owned-copy behavior: `var n: i64 = get(...)` is an
ordinary local, and assigning n does not update the stored integer.

A borrow lasts to the end of its declaring lexical block, never beyond the
hook. It cannot escape in a closure, through an ordinary helper call, or as a
borrow retained in state, cache, another entry or an output. Projections end
no later than their parent. An explicitly admitted owning retention operation
may consume the value by copying it; the borrowed access itself does not escape.

A writable borrow is exclusive. While live, a second get of its entry or a
separate set replacing that entry is rejected. Read-only borrows may coexist;
a writable borrow or replacement conflicts while any read-only borrow is live.
Creating another local alias of a writable borrow is rejected, including a
read-only alias. Read-only aliases inherit their original borrow and cannot
be upgraded to writable access or extend its lifetime. Binding a borrowed
projection as an additional live view obeys the same restrictions. Reading
an ordinary primitive field produces an ordinary value and does not lend
another aggregate view.

Whole-value assignment through a borrowed `var` replaces the value in that
same entry. It does not detach the local or retarget it to another entry.
The assigned value must have the entry's exact bound type. As with set, the
replacement is retained independently before the previous value is replaced;
a failed replacement leaves the previous entry unchanged. Reading the current
borrow as part of an assignment's right-hand side is permitted, with the
right-hand value evaluated before replacement. Derived live aggregate aliases
would conflict and are not admitted. A later statement through the same
borrow observes the replacement. A separate set of that key must wait until
the lexical borrow ends.

Known conflicts are checking errors. If const key parameters coincide only
when a graph is configured, required distinctness is validated before start
and a conflict fails construction. Disjoint control-flow lifetimes do not
overlap. Checking includes effects that could reenter a conflicting access;
unaccounted capability access cannot be hidden by an ordinary helper call.
There is no per-evaluation borrow registry. A later hook may borrow again.

For an already initialized Box entry:

```hgl
fn increment_entry(value: i64, const key: str) {
    inject global_state
    when {
        var box: Box = get(global_state, key)
        box.amount += 1
        box = Box(amount: box.amount + 1)
    }
}
```

Both writes update that entry. Replacing `var` with `let` rejects both writes;
neither change affects the entry's bound Box type. By contrast, a primitive
counter uses explicit set to retain a changed local value:

```hgl
fn increment_counter(value: i64, const key: str) {
    inject global_state
    when {
        var count: i64 = get(global_state, key)
        count += 1
        set(global_state, key, count)
    }
}
```

The ordinary-list extension supplies typed empty construction, length, indexed
reads and end growth. Structural delta storage, nullable elements and complete
replay/record bodies remain separate contracts. This extension
does not reclassify historical recordings as node cache.
