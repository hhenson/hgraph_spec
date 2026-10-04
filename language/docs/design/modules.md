# Modules and native extensions

Status: accepted module boundary.

## Boundary

Hgraph and its native extensions provide the capabilities from which language
programs compose applications. A language module is a typed, enumerable view
of one such package. It is not a general foreign-function interface.

The compiler has one mandatory module for the hgraph kernel. Its implicit
prelude supplies canonical types, temporal-shape mappings, `atomic<T>`, and the
standard operator bindings used by expression syntax. Analytics, persistence,
Kafka, web, fabric, and downstream packages remain optional.

## Module contents

A native package that supports the language supplies a descriptor containing:

- canonical language module name and version;
- compatible hgraph SDK and descriptor-format versions;
- automatically public nominal operator contracts, explicitly exported exact
  functions, and exported concrete and abstract struct declarations;
- operator implementation candidates, including explicit generic
  materializations with any retained resolver parameters, indexed by canonical operator identity,
  provider module, implementation kind, and generic signature;
- canonical types, schema declarations, generic struct-family parameters and
  constraints, and abstract-family relationships;
- specialization artifact references for requested candidates whose retained
  parameters affect body or storage types;
- target-specific native dependencies and binding metadata;
- explicit module initialization, registry installation, deinitialization, and
  registration-removal entry points;
- optional documentation and source links for diagnostics and tooling.

Exact native value functions and opaque state use the same descriptor and
lifecycle. Their phase, effect, ownership, and safety requirements are defined
in [Native interface](native-interface.md). They do not introduce a second
foreign-function or package mechanism.

The kernel descriptor additionally maps language operator tokens such as `+`
and `==` to public hgraph operator contracts. This is a syntax binding into the
same registry, not a compiler-owned implementation.

The descriptor contains declarations and build metadata, not executable user
code. Its representation is versioned and can be validated before native code
is loaded. An implementation may serialize it as declarative records.

Scalar-dependent `requires` predicates remain declarative constraints across
module boundaries. Import preserves them without approximation. An
implementation unable to represent a constraint must diagnose it; an arbitrary
resolver callback is not a substitute for the contract.

## Public declaration surface

Declaring an `operator` defines a public extension contract. A private operator
would undermine cross-module implementation discovery, so there is no
`export operator` form.

An ordinary exact `fn` is module-internal by default. `export fn` places its
signature in the public descriptor and makes it available to selective and
qualified imports. Export does not create an overload family.

A nominal `struct` is likewise module-internal by default. `export struct` and
`export abstract struct` place the module-qualified name, abstract/final kind,
parent identities, ordered field schemas, and construction metadata in the
public descriptor so downstream modules resolve the same scalar Bundle,
temporal TSB, and polymorphic family identities. The visibility rule for a
public child whose abstract parent is module-private remains to be defined.
For a generic struct, the public descriptor carries the origin, ordered type
and `const` parameter kinds, constraint IR, and unsubstituted field, default,
and parent expressions. A downstream target can therefore validate and create
the same fully applied nominal specializations without loading user code.

An `impl fn` is neither a private helper nor an independently exported exact
function. A non-generic implementation contributes a public candidate to its
operator's implementation inventory; a generic implementation contributes
only the candidates requested with `instantiate`. Applying `export`
to an `impl fn` is rejected as redundant and misleading.

The initial design has no declaration re-export. In particular, an
implementation module does not create another import route for the operator it
implements. An operator retains one defining module and canonical identity.

## Source module parts

A module may be authored across several files without creating additional
nominal modules:

```hgl
module hgraph.std part arithmetic
```

Every file in an assembled set names the same module and a unique part. The
part name is file-local ownership metadata used for deterministic compilation;
it is not a namespace, export, provider, or registration identity. All files
share one declaration and import scope and one module identity. Private declarations may be referenced across parts, and duplicate
declarations are checked across the set. Imports still precede ordinary
declarations within each physical file and never become re-exports.

The build target lists parts explicitly. The compiler neither scans a
directory nor infers a missing file. The complete decision, including source
mapping and artifact naming, is [ADR 0006](decisions/0006-multi-file-module-parts.md).

## Import and build resolution

Selective imports introduce declarations as unqualified local names:

```text
use hgraph.analytics::{rolling_mean}
```

A module alias introduces only a namespace:

```text
use hgraph.analytics as analytics

analytics::rolling_mean(price, 20)
```

There are no wildcard imports. Aliasing permits two modules to expose the same
short declaration name without conflating their nominal identities.

For either form, the compiler:

1. locates the `hgraph.analytics` descriptor through the package search path;
2. checks its hgraph and language compatibility constraints;
3. adds its public operators, exported exact functions, and exported nominal
   structs to name and type resolution;
4. records the referenced declarations and their defining modules in typed IR.

An import does not activate an implementation provider. Generated code includes
the declaration headers it uses, while the application registration plan and
link manifest are derived separately from the target's complete module closure.

## Nominal operator binding

An operator's canonical identity is its defining module plus declaration name.
Two modules may define `my_op`, but those definitions describe distinct
protocol-like contracts and own distinct implementation sets.
Every operator declaration is public.

A `fn` becomes an implementation of an operator only when it is declared
`impl fn` and its module declares that operator locally or selectively imports
exactly one such operator into the local scope:

```hgl
module my.implementation

use my.contracts::{my_op}

impl fn my_op(value: f64) -> f64 =>
    value
```

`impl fn` without an operator of that name in scope is an error, and a plain
`fn` that shares a name with an in-scope operator is a name conflict. Binding
is therefore never a side effect of an import: a `use` added for an unrelated
call cannot promote an existing private function into the candidate universe.
The uniqueness rule applies per local short name. Two selective imports that
would bind different nominal operators as `my_op` are rejected during import
resolution. An aliased module does not create an implementation binding, so
other definitions remain callable through qualified names such as
`other::my_op(...)`.

A compatible non-generic `impl fn` is an externally visible candidate of its
selected operator. A generic `impl fn` is instead a hidden source template;
each candidate requested by `instantiate op<A, ...>` is externally visible.
An `_` argument retains that slot in the candidate signature. Neither form is
directly importable as an exact function. The provider module and each
requested candidate signature are part of the descriptor; the unrestricted
generic template is not advertised as a runtime candidate.

Explicit materialization follows the same rules for local and selectively
imported contracts. The module interface preserves canonical identity,
signature, requirements and properties through registration.

Every candidate and call retains the canonical operator identity, rather than
reconstructing it from a short name. A target binding may map that identity to
a native symbol; the mapping does not create another operator.

## Cross-module retained specialization

The compiler building the consuming target owns substitution, checking and
lowering of a selected source candidate, including one supplied by a precompiled
provider. If retained parameters affect body or storage types, the provider
must package a versioned specialization artifact referenced by its descriptor.
The artifact carries body and storage IR plus the private declaration and
binding closure needed to check and lower it. It preserves provider lexical
bindings and canonical identities; private helpers gain no import visibility.

An artifact permits only the candidate pattern requested by `instantiate`.
It neither exposes the unrestricted template nor adds candidates to resolution.
`hgl check` reads it as compiler input without executing provider code; native
builds lower the same checked specialization. A missing or incompatible artifact,
an unresolved required binding, or an invalid substituted body is a checking
error, not a reason to remove a candidate or retry overload selection.

The target manifest records the provider, candidate, complete canonical bindings
and artifact fingerprint for each specialization. Distinct bindings preserve
their own types and storage layouts. Descriptor, artifact and linked provider
must agree before wiring. A signature-only retained marker needs no body
artifact; a provider without one can expose only candidates that do not require
such specialization.

## Candidate universe and provider discovery

The compiler resolves operator calls against one closed candidate universe. It
contains every implementation contributed by:

- the source modules assigned to the current package target;
- every module in the locked transitive package dependency closure;
- the hgraph kernel and selected native extension packages.

This universe is not the import graph. Imports control lexical name visibility;
the package and target graph controls implementation participation. A module
may therefore contribute an implementation of `add` without providing any
second name for `add` and without being imported at the call site.

Installed but undeclared packages are outside the universe. The compiler must
not search a machine-wide registry or plugin directory for additional
candidates: doing so would make successful builds and ambiguities depend on the
host environment.

Each descriptor advertises candidate metadata without loading executable code,
so `hgl check` can construct the same overload set as a native build. The final
link manifest includes every participating provider and generates direct
references to its initialization entry point; this both makes registration
complete and prevents static-library dead stripping. A candidate-universe
fingerprint detects a descriptor/runtime registration mismatch before graph
wiring.

## One operator resolver

The compiler owns language name resolution and type checking but does not own a
second hgraph overload algorithm. Name resolution first chooses exactly one
nominal operator. The candidates for that identity from the complete target
universe are then presented to a compiler bridge over hgraph's `TypePattern`,
`ResolutionMap`, and operator registry. The bridge returns the selected
candidate, resolved output schema, and rejected-candidate diagnostics.

Generated C++ makes the corresponding public operator wiring call. Both paths
therefore consume the same registrations and matching rules. A mismatch between
compiler prediction and generated wiring is a compiler defect and belongs in a
cross-mode regression test.

User-defined functions without a local operator binding retain exact typed
declarations and do not form overload sets. Only those declared `export fn`
enter the public declaration surface. `impl fn` declarations are registered
as their operator's candidates when non-generic; generic implementations
register only their explicit materializations. Their source bodies still determine
whether each candidate lowers as composition or a runtime node.
Exported structs enter the same declaration surface as nominal types rather
than callable candidates. Their parent relationships also enter the target's
type-registration inventory. The complete linked module closure, rather than
the set of source imports, determines the final concrete alternatives visible
to an abstract scalar or atomic value when a graph captures its type
realization. The application manifest also records the finite set of generic
struct specializations used by that target. Only those complete, validated
applications are registered; a generic origin is not a runtime value schema.

Namespace resolution is not candidate ranking. Different nominal operators
with the same short name never share a candidate set. Within one selected
operator, hgraph specificity rules choose the best implementation and an
equal-ranked overlap remains an ambiguity error. Declaration and registration
order never break the tie. Cross-module coherence rules for overlapping source
implementations remain to be designed.

## Generated module lifecycle

Modules have explicit initialization and deinitialization. Native and generated
modules obey the same lifetime contract, including when several applications
or tests share a process.

The module lifecycle has three separate responsibilities:

1. `init` attaches one module instance to an application and records its keyed
   registry installer;
2. the installer materializes that module's types, requested operator
   candidates (including `instantiate` patterns), and
   native associations for the current registry generation;
3. `deinit` removes the module's active contributions and installer intent,
   releases owned resources, and permits later unloading when safe.

`init` and `deinit` are generated module operations, not
source-level blocks. A native extension may provide reviewed resource hooks
through the same ABI. Registry installation remains replayable after reset and
must not repeat unrelated one-time initialization side effects.

Initialization is transactional and idempotent. A failed initialization rolls
back its pending contribution; repeated initialization of the same module
instance cannot duplicate candidates. Dependencies initialize first and
deinitialize in reverse topological order. Registration order is retained for
deterministic diagnostics but never resolves an overload tie.

The module manager owns registrations through an opaque handle rather than
asking generated code to erase individual candidates. Every installed
candidate carries its provider identity. Removing a handle must atomically:

- exclude the provider from future resolution;
- remove its installer intent so a reset cannot resurrect it;
- remove its exact-function and candidate registrations;
- release type associations and native resources when their leases permit.

Deactivation, deinitialization, and native-library unloading are distinct. A
wired graph or cached plan that selects module code holds a provider lease.
Deinitialization must wait or fail while such a lease is live, and unloading is
permitted only after all code and metadata references are gone. An
implementation may perform logical registration removal while retaining the
library image for process lifetime.

Registration covers every module-owned contribution: types, exact functions,
operator candidates, associations and native resources. Removal must not
leave some contributions active after the provider is deactivated.

## Native adaptor boundary

Push adaptors, services, and external-resource sinks are native capabilities.
Their module declarations expose typed function contracts, but language source
cannot provide callbacks, queue storage, thread ownership, or arbitrary
lifecycle hooks to them.

An imported adaptor owns:

- sender admission and backpressure;
- start and stop task ownership;
- service resource sharing;
- simulation versus real-time wiring;
- protocol acknowledgement and teardown.

The language function supplies temporal arguments and `const` policy values.
Dynamic behavior is represented by temporal inputs and explicit hgraph runtime
functions, not by escaping to native code.

Runtime functions may call exact native value operations admitted by a module
descriptor. Those calls are deliberately narrower than adaptor exposure: the
first slice permits non-blocking, non-throwing scalar evaluation over owned
opaque node state. It does not permit callbacks, native threads, raw pointers,
or direct adaptor implementation in HGL.

## Package manifest

A language application will eventually carry a manifest which records:

- package name and language edition;
- source roots, the modules assigned to each target, and its entry function;
- direct language module dependencies and version constraints;
- build profiles and target platforms;
- explicitly selected runtime mode and deployment packaging.

A lock file will pin descriptor packages, hgraph, native extensions, and the
native compiler toolchain inputs required for a reproducible production build.
Neither format is fixed by the initial scaffold.

## Repository boundary

This repository owns shared contracts and examples. Compiler builds, native
bindings and platform setup belong to the implementation repositories;
measured compatibility and coverage belong to the audit repository.
