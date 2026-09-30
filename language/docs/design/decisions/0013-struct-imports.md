# ADR 0013: struct imports

Status: accepted (2026-09-20).

## Context

A module must be able to export a data shape as well as behavior. Importing
that shape must preserve its nominal identity, complete layout and constraints.

## Decision

**A struct imports exactly as a function does.** `use m::{Quote}` binds the
name in the importing module for resolution. Nothing else about it is local.

**The type identity stays with the owning module.** An imported `Quote` is
`m.Quote`, not a copy under the importer's namespace. There is no second type
and no second registry entry: a name is bound, an identity is referred to.
Two modules that import the same struct name the same type, and a value
crosses between them without conversion.

**Import preserves the owner's complete description.** An implementation may
intern equivalent descriptions under one nominal identity. Import does not
require the exporting module's process to have run first and does not create
a copy of the type.

**Conflicting descriptions of one identity are rejected.** Reusing an existing
registration must not silently substitute a different layout.

**The comparison is over the whole description, not its shape.** Renaming a
field's *type* is what a version bump usually looks like, and it leaves the
field count and every field name unchanged — so kind, arity and names would
call two different schemas a match. The preflight compares each field's
realized type, the parents, abstractness and the generic arguments, and names
the part that disagrees. A recursive field is compared by the **target it
names** rather than realized, because realizing the edge would need the very
type being checked; "an owner of some named bundle" would accept an edge that
owns a different struct.

**A disagreement fails.** A diagnostic must not be accompanied by a usable
but incompatible type.

**Every reachable member is checked.** A compatible root is insufficient if a
field or parent names an incompatible type. Concurrent registration must
preserve the same agreement check; winning a registration race does not
establish compatibility.

### Imported metadata

An imported contract preserves fields, types, parents, abstractness, generic
arguments and constraints, recursive targets, optionality and defaults. It
crosses whole or is rejected with the unsupported part named. A default
contradicting requiredness is invalid metadata. Inherited defaults must retain
their declaring ancestor and any explicit override.

A generic struct's `where` requirement retains its complete meaning. Imported
and local declarations use the same constraint rules.

### Resolution

Qualified and selective imports resolve to the same owning-module identity.

**Both spellings, through one path.** `use m::{Quote}` binds the name in the
importing module and `m::Quote` names it through an alias; an unqualified name
still resolves locally first. The two forms share the binding, the arity check
and the per-argument role checks, so they cannot drift — the unqualified form
needs the `use` to bind it *and* the bare name to resolve as a type, and
missing either half makes the documented spelling fail while the alias one
works.

### Reachability and layout

Import includes every parent and field type reachable from the named struct.
Missing declarations are reported at the import; a partial layout is invalid.
Type arguments of nominal applications belong to this closure too.

Local recursive-edge rules apply unchanged to imported layouts. A cyclic
component containing an ordinary field or inheritance edge is refused; allowed
owned edges follow ADR 0012. This check must not depend on field order.

The effective layout contains inherited fields before locally declared fields.
Diamond inheritance deduplicates fields by name and preserves the declaring
ancestor's identity. An implementation could use explicit worklists and
component ordering to avoid call-stack limits on deep imported closures; the
choice of traversal algorithm does not change type identity or field layout.

### Inheriting an imported family

A local struct may inherit an imported abstract parent. Inherited fields retain
their declaring module, type identity, optionality, defaults and provenance.
The complete parent layout must be available before deriving the child's
layout; importing must not reinterpret field types in the child's namespace.

### Exports are closed under reachability

**Everything an exported struct reaches must itself be exported** — its field
types, its parents, its generic arguments, and its recursive edge targets
(ADR 0012). An importer rebuilds a struct from its layout, and a layout that
names a module-internal struct cannot be rebuilt.

This is checked **at export time**, so the error lands on the module that
broke its own contract rather than on whoever consumes it. A recursive target
must be a declared exported struct, just as every other reachable field type
must be available to the importer.

An unexported struct stays unconstrained: a module-internal leaf, or a whole
internal chain, may reference other internal structs freely. The closure rule
applies only from an exported root.


## Consequences

- A module can publish a data shape, not only behaviour. Two modules that
  import one struct exchange values without conversion.
- An exported struct's layout becomes part of its module's contract: changing
  a field changes the descriptor fingerprint, and an importer built against
  the old one is rejected rather than silently mismatched.
- A local child of an imported abstract parent joins that family's bundle
  hierarchy process-wide, and its generation advances. Polymorphic dispatch
  over the imported parent then sees the importing module's struct. That is
  the intent of publishing an abstract family, and it means a family's members
  are no longer all known to the module that declared it.

## Alternatives

**Bind the descriptor's schema identity without re-describing.** Rejected: it
requires the exporting module to have registered first, which orders two
independent compilations, and it gives the importer no way to detect that it
was built against a different layout. Re-describing detects exactly that.

**Copy the struct into the importer's namespace.** Rejected: it creates a
second type with equal fields, and the nominal identity rule
(types-and-expressions.md) says two separately declared structs with equal
fields are different types — so values would not cross between modules. It
also contradicts how `use` already works for functions.

## Transitive supply and versioning

The application supplies the transitive module closure needed by imported
parents, fields and generic arguments. Conflicting descriptions of one identity
are errors. Whether an implementation diagnoses a conflict while loading
interfaces or before registration is a tooling choice; it must reject it
before incompatible types can be used.

## Acceptance

1. A module exports a struct; a second module imports it, constructs a value,
   reads a field, and passes it to a function of the exporting module.
2. The imported type is the exporting module's type: a value built in one
   crosses to the other without conversion, and both name the same schema.
3. A recursive exported struct (ADR 0012) imports and rebuilds its edges.
4. An importer built against a changed layout is rejected with a pointed
   message, not silently mismatched.
5. Independent implementations agree tick for tick on a module pair that exports and
   imports a struct.
6. Checking validates an importing module against a descriptor without
   loading code.
