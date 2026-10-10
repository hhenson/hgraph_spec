# Native interface

Status: accepted native boundary. Target-specific authoring and ABI details
belong to implementation documentation.

The agreed authoring model is [ADR 0014](decisions/0014-native-implementation-interfaces.md):
shared declarations and selected implementations in native source. `native`
preserves ordinary temporal and `const` typing.

## Purpose

HGL is not a general-purpose foreign-function interface. Native providers
supply algorithms, resources and capabilities through checked declarations
and the shared package lifecycle.

This interface extends the package and lifecycle model in
[Modules and native extensions](modules.md). It is not a second module system.

The direction for value functions, caches and native target contracts is
specified in [ADR 0008](decisions/0008-temporal-contracts-and-target-mappings.md).

## Source native C++ functions

The legacy inline C++ authoring form is superseded by shared declarations and
selected [implementation parts](native-implementation-parts.md). Native source
and build dependencies belong to the provider, not the portable declaration.

## Descriptor is the contract

A supporting native package publishes a versioned descriptor alongside its
headers and libraries. `hgl check` reads the descriptor without loading native
code. Generated builds and scripted execution use its build metadata to link or
load the matching provider and verify the same fingerprint.

For every exposed native declaration the descriptor records:

- canonical HGL module and declaration identity;
- declaration category: hgraph operator, exact native value function, native
  constructor, or lifecycle operation;
- complete HGL parameter and result types, including generic collection-view
  patterns used only for overload selection and the complete input-only
  `signal` pattern used for payload-erased endpoint calls;
- permitted phases: wiring, start, evaluation, or stop;
- observable effects, including mutation, I/O, blocking, and allocation where
  relevant;
- value, owned, shared, or borrowed ownership and any dependent lifetime;
- whether a parameter receives its current scalar value or its live typed input
  view;
- exception and thread-safety policy;
- target-specific binding identity and native dependencies;
- module lifecycle entry points, provider identity, compatibility versions, and
  descriptor fingerprint.

An imported native declaration retains its full signature, phases, effects,
ownership, dependent lifetimes, exception policy and provider identity.
Checking must validate these before execution. The representation and format
version are implementation choices; serializing a declaration must not weaken
its contract.

## Native declaration categories

### Hgraph operators

A native temporal operator is a normal registered hgraph operator. The descriptor
exposes its nominal operator contract and provider candidates, and HGL resolves
it through the shared hgraph resolver. This is the path for graphs, nodes,
sources, sinks, adaptors, and services, including temporal sources whose
parameters are entirely fixed configuration.

An ordinary C++ scalar function is never lifted implicitly into one node per
call. A package that wants temporal use supplies and registers the corresponding
hgraph operator implementation explicitly.

### Exact native value and view functions

An exact native function is callable only in phases allowed by its descriptor.
A value parameter receives the current canonical scalar payload. A collection
`input-view` parameter receives the corresponding live `TSL`, `TSS`, `TSD`, or
rolling input. A `signal` input permits metadata access without payload
access. The native representation of the borrowed input is target-specific;
it must preserve the source's access and lifetime restrictions.

The endpoint metadata surface follows the runtime time-series rules:

| Native function | Observation | HGL result |
| --- | --- | --- |
| `valid(value)` | Top-level validity | `bool` |
| `all_valid(value)` | Validity under the shape's all-valid rule | `bool` |
| `modified(value)` | Modification in this evaluation cycle | `bool` |
| `last_modified(value)` | Last modification time | `datetime` |
| `bound(value)` | Whether the input is bound | `bool` |
| `active(value)` | Whether the input is active | `bool` |

These observations apply to the admitted scalar, bundle, list, set, map,
window and reference shapes. A target may share one native metadata interface
across shapes; it need not expose that interface's representation in HGL.

Representation erasure belongs behind the implementation boundary. HGL uses
ordinary value expressions and `delta_value` for every supported shape, not
separate erased-value accessors. Additional operations require the following contracts:

| Missing surface | Required language or ABI feature |
| --- | --- |
| ordinary value equality | generic value transport and a throwing/effect contract because user-defined equality may throw |
| current-value expressions and `delta_value` results | ordinary HGL value typing plus borrowed/dependent result lifetime |
| `reference()` | a dependent reference result whose target schema is selected from the argument |
| `hash()` | an agreed unsigned hash carrier and the throwing/unhashable contract |
| `compare()` | an HGL ordering result which represents less, equal, greater, and unordered |
| `to_string()` / `format_string()` | a generic value parameter; a `str` result by value and a `throws` policy are already admitted |
| erased output access and mutation | an output-view parameter mode with explicit mutation and lifetime rules |

Specialized collection iteration remains on typed views and HGL intrinsics; it
cannot be represented by an erased scalar result without iterator and borrowed
element contracts.

See the [native surface completion record](native-surface-proposal.md) for
accepted collection accessors and remaining decisions. In particular, the existing
TSL/TSS/TSD input patterns do not imply native access to atomic List/Set/Map
values; their signature and borrowing contracts must be supported explicitly.

Runtime pack schema inspection is a narrow borrowed capability. A helper may
inspect the current child's metadata only during the call; it may not retain
it or turn it into a payload value. Target-specific pointers or handles are
not source-level types.

Native declarations may share one canonical identity when their HGL signatures
differ. The compiler treats them as one overload family, unifies generic
collection patterns against the argument types, and records the unique selected
candidate before HGraph IR lowering. No implicit conversions or native-overload
ranking are involved: no match is an error, and overlapping matches are
ambiguous. A package should therefore publish disjoint patterns.

The generated C++ calls the declared symbol, its package-provided wrapper, or a
source-native generated function directly. It never subclasses an operator to
represent an exact helper. The direct-wiring backend does not emulate native
C++: a runtime-bearing program follows the existing generated, compiled, and
loaded image path.

Calls from required wiring-time constant evaluation, automatic temporal
lifting, and general compile-time execution are outside the first interface.
[NVAL-3](native-atomic-values.md) separately admits descriptor-permitted
ordinary executed test setup and cold materialization without runtime-only
capabilities; the source checker never executes native code. Although the
phase metadata can describe wiring, start, evaluation, and stop, the compiler
accepts a call only in a phase named by the descriptor and the agreed
canonical-value and collection-view slices are exercised in evaluation.

### Opaque native state

An opaque native type exposes no fields, inheritance, pointer operations, or
layout to HGL. It may be passed only to native functions that name the same
descriptor identity.

The planned first state bridge is an owned RAII value. Storage and objects must
be constructed during node initialization, before `start`; logical seeding,
record/replay restoration, and cache reconstruction are distinct from physical
construction. Normal destruction occurs after the stop phase, with cleanup
also required for partial initialization. A resource that needs observable
shutdown exposes a permitted stop-phase operation in addition to its destructor.

Opaque storage is not an exemption from the language's persistence contract.
Reconstructible non-recordable data belongs in HGL cache (native `State<T>`);
semantic history belongs in HGL `state` (native `RecordableState<TSchema>`) and
requires recordable types. Native nodes support both selectors with independent
storage and restore recordable state before `start` rebuilds the cache. Mixed
HGL lowering and the opaque-type lifecycle bridge remain implementation work.

Borrowed values are confined to the call or evaluation that produced them.
They cannot be returned, stored in state or output, captured, placed in a
collection, or embedded in an HGL struct. Shared and independently owned
reference forms require explicit retain/release or move/destruction contracts
and arrive after the RAII slice.

### Atomic native values

A native value that crosses a temporal port is not merely an opaque C++ type.
It must have a registered hgraph scalar identity and the public value, storage,
equality, hashing, conversion, and serialization operations required by every
context in which the descriptor permits it. HGL then exposes it as a nominal
atomic value through that canonical metadata.

The compiler does not infer those operations from a C++ class definition.
The [native atomic value contract](native-atomic-values.md) defines source
declarations, capabilities and publication admission.

## Initial safety envelope

The first native-value interface is intentionally narrow at its HGL boundary:

- value arguments and results are canonical scalar values or an opaque state
  value declared by the same module;
- collection arguments may use generic `list`, `set`, `map`, or `rolling`
  patterns only when the parameter explicitly requests `input-view` access;
- the complete input-only `signal` pattern may request `input-view` access and
  receives a common `TSInputView`; it is rejected as a value parameter, nested
  type, const parameter, or result;
- collection type and extent generics participate in compile-time selection but
  are not automatically exposed as runtime values;
- opaque state uses owned RAII storage and cannot cross a temporal port;
- evaluation functions are non-blocking; they are `noexcept` unless declared
  `throws` (descriptor policy `translated`), in which case a raise ends the
  evaluation under hgraph's node error model (ADR 0009);
- mutation is restricted to an explicitly identified state argument;
- raw pointers, lifetimes, callbacks, variadic calls, and open C++ templates are
  not representable in the HGL signature or descriptor; a local C++ body is
  still real C++ and remains the author's responsibility;
- an open C++ template is still exposed only through an explicit specialization
  or wrapper; generic HGL input-view patterns erase to reviewed non-template C++
  view types;
- native declarations do not participate in implicit conversions;
- descriptor and loaded-provider fingerprints must agree before wiring.

These restrictions can be relaxed individually when a real core or extension
migration requires them and their HGL-facing semantics are defined.

## Desired HGL experience

A small helper owned by an HGL module can be implemented directly. The HGL
signature remains the public contract and the C++ projection states the exact
native ABI used by the generated call:

[Native example 3](https://github.com/hhenson/hgraph_spec_audit/blob/main/examples/documentation/language/docs/design/native-interface.md#example-3)

For a separately built native package, an imported descriptor supplies the
same call-site contract. For example, a package may expose a non-throwing
scalar update function so an HGL node can be written as:

```hgl
use acme.stats as stats

fn smooth(value: f64, const window: i64) -> f64 {
    state previous: f64 = 0.0

    when modified(value) && valid(value) {
        previous = stats::update(previous, value, window)
        return previous
    }
}
```

An imported overload family uses the same call syntax. Here `len` is a direct
native view operation, while a public temporal `len_` operator can call it from
its runtime implementation:

```hgl
use hgraph.native as native

impl fn len_<T, const size: i64>(value: list<T, size>) -> i64 {
    when {
        return native::len(value)
    }
}
```

`T` and `size` select the list-view overload. The body does not need either
value: the selected C++ overload reads `value.size()` from the live input view.
This is the important distinction between a generic required to instantiate or
select a callable, a marker retained only as part of a type, and runtime
information explicitly available through a native view.

Payload-erased behavior uses the same direct-call model without a generic
overload family:

```hgl
use hgraph.native as native

fn observe(value: signal) -> datetime {
    when {
        return native::last_modified(value)
    }
}
```

The node accepts any admitted time-series shape. The helper borrows metadata
without exposing its payload; `signal` does not grant value access.

The source spelling and inference rules for an imported opaque state type are
not settled, so this record does not invent an example for them. The native
implementation must stop at that design question if existing nominal type
syntax is insufficient.

## Producing descriptors

Native libraries supply reviewable module interfaces. An implementation could
generate them from annotated source or checked declarations. The generated
interface must carry the same contract and dependency identity; reflection
must not broaden what HGL admits.

## Module lifecycle and ABI

A provider initializes transactionally, installs its contributions through an
owned registration, and deinitializes in reverse dependency order. Graphs,
plans, call targets and metadata retain a lease on their provider.

Initialization and deinitialization are idempotent. Before activation, a loader
checks module identity and interface compatibility. An implementation could
use a versioned function table and a fingerprint of the semantic interface;
its layout, symbol names and digest encoding are not language requirements.

Logical removal and physical unloading are distinct. Removing a provider
prevents future selection, but unloading must wait until no reachable code or
metadata depends on it. An implementation may retain removed images until
process exit.

## Acceptance

The native boundary proves:

- descriptor-only `hgl check` without loading its library;
- one canonical scalar value function used inside a runtime node in an AOT
  module;
- one owned opaque state value constructed at startup, mutated during
  evaluation, and destroyed after stop;
- rejection of the same calls in an unpermitted phase;
- rejection of a borrowed value that escapes;
- native calls bound to the selected contract without per-tick name lookup;
- a native declaration exported, imported and executed through its selected
  provider;
- interpreted and compiled execution with identical ticks;
- descriptor/provider fingerprint mismatch before graph wiring;
- failed activation rollback and provider removal without stale registrations;
- an installed-SDK consumer build, not only an in-tree test.
