# ADR 0012: recursive struct fields

Status: accepted (2026-09-19). Local and imported recursive structs obey the
same edge rules; see [ADR 0013](0013-struct-imports.md).

## Context

A list node, a tree and an expression family are ordinary value shapes, and
recursive fields were always meant to be part of HGL. The first structured-
value slice deferred them ("There are no ... self-recursive fields in the
first slice") and they have stayed on the open-question list since.

An implementation could represent a recursive atomic field as an optional
owner of the target value. The representation must preserve value semantics,
including independent copies; it does not alter the recursive-edge rules.

Two properties of HGL make recursion more than a matter of removing the
check:

- one struct declaration gives both a scalar schema and a temporal shape, and
  temporalization is recursive. A temporal `Node` with a temporal `next: Node`
  is a bundle that never ends;
- a required recursive field has no finite value.

## Decision

1. **A field may name its own struct, or a struct of the same module that
   leads back to it.** That field is a *recursive edge*. No new syntax is
   introduced: a recursive edge is an ordinary field declaration.

2. **A recursive edge is optional.** It is declared `= null`. A recursive edge
   without a null default is an error, because no value of the struct could
   be completed. A value is therefore always a finite tree; there is no way to
   construct a cycle.

3. **A recursive edge is an atomic boundary, and says so.** Its declared type
   is `atomic<T>`:

   ```hgl
   struct Node {
       value: i64
       next: atomic<Node> = null
   }
   ```

   This is the existing rule that an atomic boundary belongs in the
   declaration, applied where it is forced. The marker is erased when the
   canonical scalar schema is derived, so `const Node` and `atomic<Node>` are
   the plain recursive value; temporal `Node` is a finite bundle whose `next`
   is one endpoint carrying a complete `Node` or null. A recursive edge not
   written `atomic<...>` is an error that names the fix.

4. **Generic structs recurse on themselves only.** Inside `Tree<T>`, a
   recursive edge is `atomic<Tree<T>>` with the struct's own parameters
   unchanged. `Tree<list<T>>` inside `Tree<T>` is rejected: it denotes an
   unbounded family of specializations.

5. **Mutual recursion is confined to one module** and is lowered as one batch.
   Every edge that closes a cycle obeys rules 2 and 3. Module imports are
   acyclic, so a cycle cannot cross modules.

6. **Recursion through an abstract parent is a recursive edge.** In
   `struct Add: Expr { lhs: atomic<Expr> = null }` the edge's target is the
   closed family that contains `Add`. It obeys rules 2 and 3.

7. **Values are compared, hashed, ordered and copied through their whole
   depth**, by hgraph's existing `Owned` operations. `fields(U)` and
   `field_type(U, name)` report a recursive edge as its declared type.

8. **Recursion through a container stays rejected** —
   `children: atomic<list<Node>> = null` — until hgraph can express it. The
   registry's recursive declarations accept only a direct owned edge to a
   member of the batch; `list<Self>` needs `Self`'s schema before it exists.
   This is recorded as an RFC ask on hgraph, not worked around in the
   compiler.

## Clarifications (2026-09-19)

Implementing the resolver raised four questions the rules above did not
settle. The owner decided them as follows; the rules are read with these
answers.

- **Rule 4 admits whatever hgraph can register.** A generic struct may join a
  wider cycle (`Tree<T>` and `Forest<T>`), recurse through its abstract parent
  (`struct Add<T>: Expr<T> { lhs: atomic<Expr<T>> = null }`), or be reached
  from a non-generic struct, provided the cycle reaches finitely many
  specializations: every generic argument on an edge is a parameter of the
  declaring struct or mentions none. `Tree<list<T>>` inside `Tree<T>` stays
  rejected.
- **A cycle through inheritance is rejected.** A parent's field that names its
  own descendant (`abstract struct Base { child: atomic<Leaf> = null }` with
  `struct Leaf: Base {}`) cannot be registered, because hgraph declares a
  parent before its children and a recursive batch cannot name one of its own
  members as a parent. What hgraph cannot implement, HGL does not allow.
- **Recursion through another struct's generic argument is a container edge**
  under rule 8 (`boxed: atomic<Box<Node>> = null`): the argument's schema would
  be needed before it exists.
- **Rule 2's null default is permanent.** A descendant may not replace a
  recursive edge's null default with a value.

## Consequences

- Every type operation must terminate on admitted recursive types, including
  capability checks, temporalization, constraint reflection and substitution.
- A recursive edge retains its target's nominal identity rather than expanding
  the target indefinitely.
- Module interfaces preserve recursive targets, including mutual recursion.
- `delta<S>` needs no new rule: a recursive edge is atomic, so a delta replaces
  the field whole or omits it.

## Alternatives

- *An implicit atomic boundary*: `next: Node = null`, with the compiler
  stopping temporalization silently. Shorter, and the only possible meaning;
  but it makes the temporal shape differ from what the declaration says, in
  exactly the place an author most needs to see it. Rule 3's diagnostic can
  name the fix, so the explicit form costs one error message.
- *Allowing a required recursive edge and rejecting the constructor.* It moves
  a declaration error to every use.
- *A new `owned<T>` type constructor in source.* It would expose a storage
  choice hgraph makes on the language's behalf, and invents syntax for a
  question `atomic` and `= null` already answer.
- *Leaving recursion to native types.* A tree then cannot be declared,
  constructed or matched in HGL at all.

## Acceptance

- resolver tests: a direct edge, a same-module mutual pair, an edge through
  an abstract parent and a generic self edge are accepted; a required edge, a
  non-atomic edge, a changed generic argument, a container edge and a
  cross-module cycle are each rejected with a diagnostic that names the rule;
- the existing rejection test is replaced, not deleted;
- a temporal use of a recursive struct has the finite bundle shape of rule 3
  across independent implementations;
- construction, equality, hashing and a three-deep value round-trip inside
  the graph in the parity corpus, observed through scalar `eval` results,
  with identical ticks from direct wiring and generated C++;
- a module descriptor carrying a recursive struct is written, validated by
  `hgl check` without loading code, and imported by a second module
  (`examples/struct-imports/`, ADR 0013 slice 8: the edge's mandatory
  `= null` is the one default the catalog carries, precisely so this closes);
- an example under `examples/` and a user-guide section.
