# Native implementation interfaces

Status: agreed model.

The HGL declaration owns typing, temporal role, borrowing, effects and errors.
A library selects one native implementation of the shared contract. An
implementation could generate a checked adapter or a trait for native code to
implement. Dependencies stay in the native library build; there is no second
per-function symbol manifest.

- `native fn`: time-series inputs and result; `const` parameters remain fixed
  configuration. All-`const` inputs do not make the function value-level.
- `native const fn`: ordinary value-function typing and no independent ticks.
  A scalar helper cannot implement a temporal declaration merely because its
  payload types match.
- Use the same checked function contract for HGL and native bodies. Binding
  selects an implementation; it does not define another function kind.
- Borrowed endpoint access must be explicit. Do not reinterpret ordinary
  collection values as live input views. Its source spelling remains pending.
- Resolve overloads and normalize substituted REF types before binding checks.
  Preserve the selected contract in IR; select the provider when building
  its library. No per-tick name lookup.
- Reject missing members, convertible-but-wrong signatures, incompatible
  temporal roles and error policies. Native compilation checks ABI shape;
  language checks retain phase, validity and lifetime obligations.

The scalar contract for the initial port is:

```hgl
module hgraph.native
native const fn bit_and(lhs: i64, rhs: i64) -> i64
```

A native implementation of this helper computes `lhs & rhs`. Its temporal
caller retains responsibility for activation and publication; the helper adds
no node or tick.

Acceptance: preserve the reasoned/Python/C++/Rust bitwise traces in
[stdlib cases](../../../compiler/stdlib/cases.json), including missing and duplicate ticks.
Compile-fail cases cover wrong argument/result types, temporal-to-scalar
binding, missing members and borrowed-view escape. Keep generated interface
checks separate from runtime trace evidence.

## Implementation parts and injectables

Keep the callable signature in the shared interface:

```hgl
module std.native
native fn filter(value: i64, const limit: i64) -> i64
```

Select a target part in the library build. Its body describes implementation
requirements. `{}` means a native graph; `{ when; }` means a native node:

```hgl
module std.native part cpp_impl
native fn filter(value: i64, const limit: i64) -> i64 {
    inject out, logger
    start;
    when;
    stop;
}
```

The implementation completes exactly one matching declaration; it is not an
overload. Match parameter names, types, constness, generics, result and exception
policy. Reject duplicate declarations, selected implementations or hooks.
`start;` and `stop;` require `when;`. Part names have no target-selection magic.
An unnamed shared interface may accompany named parts.

`native const fn` remains a value helper. Its selected body may request logger
or clock but cannot declare node lifecycle, output or scheduler ownership.
C++ and Rust parts may request different services. After selection, requirements
silently propagate through value-helper callers, transitively and without
duplicates. Observable behaviour remains the shared function's obligation.
Unavailable services are checking errors.

`out` denotes the declared temporal result, not another output. Constructed
nodes own their output, scheduler and lifecycle; graph-construction callers do
not borrow those capabilities. Nested nodes follow engine activation/teardown.
Native constructors/destructors and Rust `Drop` do not replace start/stop.

Selected injectables must be available at the call site. A temporal graph or
node contract cannot use a scalar value-call ABI merely because payload types
match. Borrowed-TS helper syntax and temporal provider binding remain separate
design questions.

A node calls a value helper directly during evaluation:

```hgl
fn describe_each(value: i64) -> str {
    when { return describe(value) }
}
```

The helper receives the current scalar value and borrows the enclosing node's
logger. Its string result is published by the enclosing `return`; the helper
creates no node or output. Its requirements contribute to the node's contract.
Calling a temporal `fn`, native or HGL, instead belongs to graph construction
and is rejected inside `when`. A helper mutating its caller's output requires
explicit borrowed access; that spelling remains unsettled.

[Implementation-part rules and acceptance cases](native-implementation-parts.md)
cover matching, selection, ownership and lifecycle diagnostics. Binding-shape
checks and runtime trace conformance are separate obligations.

## Proposed scalar eval buffer capabilities

[ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md) proposes two
node-scoped typed injectables for normal HGL replay/record operator bodies.
Their native methods only inspect immutable dense slots or begin/append an
owned capture. HGL owns scheduling, publication and capture decisions.
These capabilities cannot be passed as native value-helper arguments or
propagated through value functions in this slice. This adds no generic
resource type, borrowed-TS helper ABI or temporal native provider.
The record specifies the methods and their type, phase, lifetime and failure
rules.
