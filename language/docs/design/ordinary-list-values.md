# Ordinary list values

Status: proposed ordinary-value operations extension, 2026-10-03.

An ordinary `list<T>` is an ordered, finite collection of values of one
concrete ordinary type T. It is unbounded in its type, so its length may
change. `list<T, N>` has exactly N elements; N is a nonnegative constant.
`list<T, unbounded>` and `list<T>` name the same unbounded type. Fixed size
remains part of canonical type identity. Access authority is separate:
[value mutability](value-mutability.md) determines who may change a value.

This contract admits typed empty construction, length, indexed reads and
end growth. It does not define nullable elements, structural delta storage,
replay element mappings or recording-specific operations. Every element is a
complete ordinary T. Harness `_` is not an ordinary element. Structural
deltas obtain their ordinary element type through the separate
[delta type contract](ordinary-delta-types.md), not through list operations.

## Typed empty construction

Use the existing empty list literal with an expected concrete list type:

```hgl
var samples: list<i64> = []
let no_samples: list<i64, 0> = []
```

The literal obtains T and fixedness from that expected type. For unbounded
lists it constructs a length-zero value; for a fixed list it is admitted only
when N is zero. This also applies where an already typed owning assignment,
ordinary parameter or constructor field supplies the exact expected list type.
The literal is a new owning value, not an alias of some shared mutable empty
list. A `let` binding makes it recursively read-only; `var` makes it writable.

Without an expected concrete element type, `[]` is a checking error. Later
calls do not infer an earlier empty list's type. An empty literal does not
supply a default element for a positive fixed size. Homogeneous nonempty
constant list literals retain their existing construction rules. This adds
neither a type-name constructor call nor general runtime-expression list
literal construction.

## Length and indexed reads

`len(values)` returns an `i64` giving the current element count. It is zero
for an empty list and N for a fixed-size list. A list's admitted length must
fit in nonnegative `i64`; construction or growth that would exceed that range
fails before changing the list. The `unbounded` type-size sentinel is never a
runtime length.

`values[index]` uses a zero-based `i64` index. It requires
`0 <= index < len(values)` and reads the element of exact type T. A negative
index does not count from the end. An out-of-range read is a bounds failure,
not absence, a default element or an invalid time-series child.

Indexing is an ordinary projection and preserves the source's access and
lifetime restrictions. It does not itself retain an independent capture.
Initializing an owning local from an owning list element follows the existing
independent-copy rule. An aggregate projection from a borrowed global entry keeps its borrow
provenance: it cannot initialize an owning local by silently copying, and
additional live aliases obey the existing borrow restrictions. A primitive
indexed read produces an ordinary owned value, just as a primitive field
read does. Keeping that primitive does not keep an aggregate borrow or prevent
a later push through the original writable access. Passing an element to an
explicitly owning retention operation copies it independently. A writable
indexed projection may receive an admitted content operation: for an owning
`var outer: list<list<i64>>`, `push(outer[0], 2)` grows its first child. This
does not replace the indexed element and introduces no additional local alias.

`len` and indexing do not consume elements or change the list. They do not
publish, schedule or interpret time. This contract adds no indexed element
replacement syntax, slicing, negative indexing or nullable read result.

## End growth

Use the existing receiver-first spelling:

```hgl
var samples: list<i64> = []
push(samples, 10)
push(samples, 20)
```

`push(values, item)` is a statement with no result value. It requires an
unbounded ordinary list reached through writable owner access or an exclusive
writable global-entry borrow. The item must have the list's exact element
type, under existing ordinary type checking. Fixed-size lists reject push,
even a zero-length fixed list. A read-only receiver is a checking error.

A successful call appends one independently retained copy of item at the old
length. The length increases by one; existing elements keep their order and
values. Retention is recursive: changing item or any of its children later
cannot change the appended value, and changing the receiving list cannot
change item. The source remains usable. The item is read and retained before
the list is extended, so `push(values, values[0])` is admitted for a nonempty
writable unbounded list and retains the original first element. This direct
read creates no additional lexical alias.

A writable global-entry borrow grows that same entry. Obtaining the borrow
and accessing its length do not require an owning copy of the whole entry.
Push does not require an independently owned replacement of the whole list;
allocation, capacity growth and physical sharing remain representation
choices. No allocation-free insertion or particular storage layout is
promised. Ownership independence applies to the inserted payload regardless
of those choices. Retention and any required capacity acquisition complete
before the new element becomes part of the logical sequence. If either fails,
the list keeps its previous length, elements and element values. Such a failure
propagates through the applicable value-operation error contract; a node
evaluation uses the translated node error contract. This failure guarantee is
a requirement of this extension, not an assumption about a representation.

The first argument distinguishes this ordinary-value operation from
`push(out, item)` on an injected temporal output. Ordinary push has no output
publication effect and does not classify a function as a runtime node. It
executes directly in value functions, on ordinary values during wiring, and
inside start, evaluation or stop hooks. It cannot mutate a temporal input or
be lifted into a mutating graph operation on someone else's connection.
Ordinary parameters and const configuration remain recursively read-only.

## Errors and boundaries

Missing type context, wrong element or index type, fixed-list growth,
read-only mutation and forbidden borrowed aliases are checking errors.
Bounds and length-range errors fail when the operation is evaluated:
constant evaluation fails checking, wiring-time evaluation fails construction,
and a node hook follows its applicable error contract. An operation whose
precondition fails does not alter the list. This does not roll back earlier
successful operations or change the node error rule that earlier writes stand.
The push failure guarantee also covers failure to retain the item or obtain
capacity; it does not roll back unrelated earlier effects.

Whole-list assignment retains its existing exact-type and ownership rules;
it is not implicit conversion between fixed and unbounded lists. This extension
does not admit pop, clear, truncation, middle insertion, bulk extension,
indexed replacement, mutation during a borrowed traversal, new escaping
aggregate aliases, or an explicit copying helper. Each needs its own contract.

[Ordinary list cases](../../../runtime/cases_ordinary_lists.md) state expected
acceptance, rejection and observable results. The
[source example](../../examples/ordinary-list-values.hgl) uses only ordinary
values; it is not a complete replay or record implementation.
