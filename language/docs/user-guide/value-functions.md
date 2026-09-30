# Value functions and lifting

`const fn` computes a value without independently scheduling a node:

```hgl
const fn scale(value: f64, factor: f64 = 2.0) -> f64 =>
    value * factor
```

The arguments are values for this invocation, not values that must remain
unchanged throughout a graph run. `const fn` does not mean compile-time-only
or pure. A parameter explicitly marked `const` still requires fixed scalar
configuration. Defaults are also allowed on ordinary value-function parameters.

## Selection comes before lifting

A temporal `fn` and a value `const fn` can have the same name. They remain
distinct declarations, not duplicate implementations of one execution role.

| Call context | Normal selection |
| --- | --- |
| Graph composition | Select the temporal `fn` if declared; otherwise select the `const fn` |
| `when`, `start`, `stop`, or another `const fn` | Select the value `const fn`; temporal calls are unavailable |
| `eval` | Select the temporal `fn` if declared; otherwise test the lifted `const fn` |

`const(scale)` explicitly selects the value declaration, overriding the first
and third rules. It does not itself construct a node or call the function:

```hgl
fn scale(value: f64, const factor: f64 = 2.0) -> f64 {
    when {
        # Inside when, scale implicitly selects the const fn above.
        return scale(value, factor) + 10.0
    }
}

fn normal(value: f64) -> f64 {
    scale(value, 3.0)                 # Uses the temporal definition.
}

fn value_version(value: f64) -> f64 {
    const(scale)(value, 3.0)          # Selects the value definition, then lifts.
}
```

Inside `when`, calling `scale(value, factor)` already selects the value
function; the explicit wrapper there is optional. Selection is independent of
declaration order. An incompatible signature or an error in the selected
temporal definition is an error, not permission to switch to the value role.
Two ordinary declarations of the same name *and role* remain a duplicate-name
error; this feature does not introduce a new overload-ranking algorithm.

## What lifting does

In graph composition, a selected value function with temporal arguments is
wrapped in a runtime node with the normal `when { ... }` policy:

- temporal arguments become inputs;
- scalar arguments, including omitted defaults, remain fixed configuration;
- any input modification activates evaluation;
- all input endpoints must be valid;
- the value result becomes an output tick, without implicit deduplication.

The rule uses ordinary `valid`, without the additional child checks of `all_valid`. A value function
that returns no value is lifted as an outputless node. Borrowed ranges or
endpoint handles do not become owned outputs by being passed through a helper.

With only scalar arguments, a value call executes directly. It does not create
a source node with invented timing. Inside evaluation or lifecycle code, it
also executes directly; the containing node owns scheduling and output.
For different activation or validity rules, write the temporal wrapper
explicitly. There is no separate `lift` configuration language.

## Testing the value version

`eval` applies the same lifting rules as graph composition:

```hgl
test scale_values {
    assert eval(const(scale), value: [1.0, _, 3.0], factor: 3.0) == [3.0, _, 9.0]
    assert eval(const(scale), value: [1.0, _, 3.0], factor: [_, 4.0, _]) == [_, 4.0, 12.0]
    assert eval(const(scale), value: [1.0, 2.0]) == [2.0, 4.0]
}
```

The second case cannot evaluate on the first tick because `factor` is invalid.
Its second tick uses the retained `value`, and the third uses the retained
`factor`. `_` means no input tick; it is not a request to invalidate an input.

`eval(scale, ...)` tests the temporal definition when present. For a name with
only a `const fn`, the wrapper is unnecessary. At least one argument must be a
tick sequence when evaluating a value function; use a direct call/assertion to
test an all-scalar invocation. The harness recognizes sequence literals
as temporal inputs and scalar expressions as configuration; its existing
structural replay limitations remain in force.

## Value-function boundary


Value functions compute with scalar values rather than time-series ports.
Structural and generic value-function forms require defined type and lifetime
contracts; an implementation must diagnose forms it cannot represent.


A value function cannot declare `when`, node state or lifecycle hooks, or inject
its own output or scheduler. It may `inject logger` or `inject clock` from its
runtime call context. Calls silently add those requirements to callers,
transitively and without duplicates; callers need not repeat the declarations.
Wiring-time capability adapters remain unsupported. See
[Native interfaces](../design/decisions/0014-native-implementation-interfaces.md#calling-from-a-node).
It cannot treat its value parameters as live temporal endpoints. Native helper
phase permissions propagate through value calls: an evaluation-only native
dependency cannot be hidden inside a helper and then called during wiring,
`start`, or `stop`. This does not add native cache, ownership, or effect syntax.
