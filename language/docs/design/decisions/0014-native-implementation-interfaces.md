# ADR 0014: Native implementation interfaces

Status: accepted authoring model.

## Contract

An HGL native declaration defines the language contract. It contains no target
body, header, crate path or symbol map. Native providers implement the same
contract in their own source files and builds.

`native` changes the implementation language, not the typing rules:

- `native fn` is temporal. Non-`const` parameters are time-series inputs and
  its result is temporal.
- A `const` parameter is fixed configuration. Even an all-`const` temporal
  function can produce ticks; it does not become a value function.
- `native const fn` is value-level. It has no independent activation or output
  ticks. Parameters and results follow ordinary `const fn` typing.
- A borrowed input must be explicit in the language contract. An ordinary
  value parameter cannot silently acquire endpoint access because it is native.

Use one checked function contract for HGL and native implementations. Function
and parameter `const`, overload selection, capability requirements and phase
checks have the same meaning. Binding consumes that contract; it does not
introduce a separate native type system or a third function kind.

For example, this contract admits a scalar implementation:

```hgl
module example.native

native const fn bit_and(lhs: i64, rhs: i64) -> i64
```

Removing `const` changes the contract to a temporal function. A scalar callable
must then fail binding validation. Neither parameter names nor scalar payload
types erase that distinction.

## Implementations

Derive binding checks from the shared HGL contract: resolved parameter and
result types, temporal role and declared failure behavior must match. Native
implicit conversions cannot satisfy a mismatched signature. Overloads retain
their HGL identities; compiler-generated candidate ordinals are not public API.

The compiler retains the selected contract through checked IR. The native
library build selects its provider; calls use that static implementation. No per-tick name lookup or
registration is introduced. Lifetime, validity, phase and error rules remain
part of the HGL contract; a matching native signature alone does not prove them.

## Outputs and capabilities

Separate the shared signature from target implementation requirements. The
[implementation-part rules](../native-implementation-parts.md) define matching
and acceptance cases. A temporal `{}` is a graph; `{ when; }` is a node.

Shared interface:

```hgl
module example.native
native fn accumulate(value: i64) -> i64
native const fn describe(value: i64) -> str
```

Selected target part:

```hgl
module example.native part cpp_impl
native fn accumulate(value: i64) -> i64 {
    inject out, logger
    when;
}
native const fn describe(value: i64) -> str {
    inject logger
}
```

Binding must preserve the checked contracts. Value-helper access to logger or
clock requires the selected provider to supply it in an admitted phase.

An HGL implementation requests capabilities in its body. The checked contract
records capability requirements for both forms, including descriptor imports.
They add no caller-supplied argument, temporal input or activation dependency.
A call silently adds the callee's required injectables to the caller's list.
Compute this transitively, independent of declaration order, and deduplicate
explicit and inferred requests. Missing `inject` in the caller is not an error;
an unavailable capability or forbidden phase still is.

The shared declaration owns types and observable behaviour. The selected
implementation part owns injectable requests and lifecycle hooks. One part
may request `out, logger` while another requests only `out`; each binding uses
its selected requirements. Calls silently inherit selected value-helper
requirements. A constructed node owns its own capabilities. Target selection
may expose an unavailable service, which is a checking error. It cannot relax
promised observable behaviour. Provider-private allocators and scratch storage
remain native implementation details.

| Request | Capability type | Ownership and access |
| --- | --- | --- |
| `out` | `Output<T>` from the temporal result `-> T` | Current node; evaluation writes |
| `logger` | `Logger` | Supplied context; logging only |
| `clock` | `EvaluationClock` | Current run; read-only, admitted runtime phases |
| `scheduler` | `Scheduler` | Current node; admitted runtime phases |

These are language capability types, not payloads to temporalize. Their public
type spelling and target binding remain separate design work. `out` designates
the existing result, never an additional output. An outputless function cannot
request it. For nested collections, its shape is the complete result schema.
Existing validity, delta and write-order rules still apply.

`const fn` means non-temporal, not pure. It may request a logger when the call
context supplies one; otherwise the call is rejected. Injection alone must not
classify it as a node. It returns its value directly and cannot request its own
temporal `out`, scheduler or node state. Access to a caller's node capabilities
needs an explicit ownership contract; that extension is not settled here.
Clock access requires a runtime context and an admitted phase, not merely a
`const fn` declaration. No injection implicitly creates a run or node.

Capability access is borrowed for the call. It cannot escape in a result,
state or cache. Check requirements at each call, including transitive calls,
and preserve them across imports. Do not constant-fold or reorder effects
merely because the function is `const`.

A temporal provider publishing through `out` still has the declared temporal
result even if its native hook returns no value. A value helper returns its
ordinary scalar result. Complete-value temporal provider binding and the full
lifecycle ABI remain separate contracts.

### Calling from a node

```hgl
fn describe_each(value: i64) -> str {
    when {
        return describe(value)
    }
}
```

During evaluation, `describe` receives the current scalar value and borrows the
enclosing node's logger. It returns a string; the enclosing `return` publishes
it. No helper node or output is created. The node omits `inject logger`: the
compiler silently adds the selected helper implementation's requirements.

A call to `native fn`, like a call to HGL `fn`, composes or wires a temporal
computation during graph construction. It is not a direct call inside `when`.
A helper that mutates the caller's output needs explicit borrowed access;
the spelling and checking of that access remain unsettled. Its scalar result
never implies ownership of the caller's temporal output.

### Acceptance cases

These cases specify expected binding, capability and call behavior. Temporal
provider binding and borrowed-output access retain their separate contract
boundaries.

| Case | Expected result |
| --- | --- |
| `const fn` and `native const fn` with `i64` argument/result | Scalar types in both; no temporal argument introduced by binding |
| `fn` and `native fn` with `i64` argument/result | Temporal types in both; a parameter marked `const` remains configuration |
| Temporal `-> i64` with `inject out` | One `Output<i64>`; no extra output or activation dependency |
| Outputless or value function requests `out` | Diagnostic; a scalar return is not a temporal output |
| Value helper requests logger in a supplied logging context | Direct value call plus logging; no extra node or tick |
| Caller omits `inject logger`, including through a helper/import | Silently infer one logger requirement on each caller |
| Inferred capability has no valid provider in the call context | Diagnostic before emission |
| One part requests `out, logger`; another requests only `out` | Same signature; each selected implementation part supplies its requirements |
| Provider requests a capability absent from its selected HGL implementation part | Binding diagnostic; internal runtime machinery needs no HGL declaration |
| Target cannot supply a required capability | Binding diagnostic; no silent fallback |
| Scheduler request without a node, or clock without runtime context | Diagnostic; no implicit owner |
| Capability used in a forbidden phase or retained after the call | Diagnostic |
| Node returns an identity logging helper's result for input ticks `2, _, 2` | Two helper calls and output ticks `2, _, 2`; no tick on the idle cycle |
| `describe_each` receives input ticks `2, _, 2` | Two direct helper calls using the same node's logger; two string ticks, no tick on the idle cycle |
| Temporal HGL/native function called inside `when` | Same diagnostic; no graph wiring during evaluation |
| Nested output receives two different child writes in one evaluation | One accumulated delta under the existing output rules |

Derive runtime observations before execution and apply the existing
[conformance acceptance policy](../../../../runtime/conformance.md#accepting-an-expectation).
Check values, ticks, effects and owner identity. Source diagnostics and native
binding checks are separate from runtime trace conformance.

## Dependencies

HGL consumers import the shared interface. A package selects its implementation
once. Native imports and dependencies stay in that library's ordinary build. The build exports generated binding information for
consumers, including the provider entry point and required link dependencies.
There is no separately authored per-function configuration or symbol manifest.

Generated descriptors are compiler/build output, not a second authored copy of
the interface. Checking an interface must not load its native library.

## Acceptance

| Scenario | Required result |
|---|---|
| Value `bit_and(6, 3)` | `2` |
| Temporal `bit_and` bound to scalar arguments/result | Compile error |
| All-`const` parameters on temporal function | Retain temporal result/activation |
| Wrong argument or result type | Native compile error, including convertible types |
| Missing member or incompatible overload | Native compile error |
| Declared result<T,error> bound to an ordinary-T implementation | Reject incompatible canonical result contract |
| Value helper called from a node | Existing activation and output tick rules |
| Borrowed input retained beyond its call | Rejected by lifetime contract |
| Provider missing or ambiguous | Diagnostic before executable generation |
| Nested REF introduced by substitution | Normalize before binding checks |

Target-specific ABI exception validation is separate from portable HGL source
semantics. Recoverable Err outcomes follow the declared result type; strict
operational failures follow their canonical execution-failure contract.
Neither is inferred from a retired source exception clause.

Record expected results before execution. Native binding must preserve scalar
operator values, ticks and effects.
Collection-view spelling and its migration must be settled before replacing
existing view helpers. Opaque state and recovery require their own contracts.
