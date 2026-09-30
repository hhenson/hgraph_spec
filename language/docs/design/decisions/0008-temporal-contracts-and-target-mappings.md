# ADR 0008: temporal programming, value functions, and target mappings

Status: accepted design direction. Value-function roles, cache semantics and
native target contracts are defined below. Unresolved syntax is identified
separately from agreed behavior.

## Context

HGL needs to express the core node and graph library without binding its
language contracts to one implementation's C++ spelling. The immediate cases
are reusable value-level helpers, recoverable node-local caches, and native
types with explicit lifetime operations. An alternative C++ engine should be
able to share some mappings while replacing others; a future Rust or Zig
target may require different representations entirely.

A target preserves the shared type, operator, lifecycle and record/replay
contracts independently of its native language or storage representation.

## HGL is a temporal programming language

HGL is a temporal programming language for expressing computations over values
that evolve through time. It combines temporal computations with value-level
functions, explicit state, and native implementations.

Change, validity, activation, and history are part of the programming model.
Graphs and nodes describe how temporal computations compose and execute; they
are not competing source-language categories. Native or imported types remain
nominal atomic values, like `i64`, `f64`, and `str`, rather than becoming
time-series types by virtue of their declaration. Their use in a temporal
parameter introduces the temporal context.

## Value-level functions

Use `const fn` to declare a non-temporal function. It executes directly on
values, or on explicitly admitted views, and has no independent activation,
output ticks, or graph topology. Its arguments and result are value-level;
they are not recursively temporalized as ordinary temporal parameters are.

An ordinary `fn` retains the current temporal callable model. Calling it at
wiring time either composes existing operations or wires a runtime node,
depending on its body. In particular, an ordinary `fn` containing `when`
does not become a value function merely because its body runs at tick time.

| Call context | Temporal `fn` or operator implementation | Value-level `const fn` implementation |
| --- | --- | --- |
| Graph construction | Compose topology or wire a node | Execute on available values, if its phase/effect contract permits wiring-time use |
| Runtime node evaluation | Cannot wire or invoke a temporal computation as a value call | Execute on current values or admitted views; the caller owns activation and output |
| Node lifecycle hook | Cannot introduce topology | Execute only if the helper is admitted in that lifecycle phase |

The marker does not mean compile-time-only, constant folding, purity, immutable
arguments, or absence of side effects. A native helper may mutate an explicitly
permitted cache argument. Mutation, allocation, I/O, borrowing, and allowed
lifecycle phases need their own contracts; `const fn` does not authorize them.
A value function cannot declare its own node state or contain `when`, `start`,
or `stop` blocks. Its calls remain value-level. The agreed
[capability contract](0014-native-implementation-interfaces.md#outputs-and-capabilities)
allows context-supplied services such as logging without creating a node;
ownership and phase checks govern access. Value helpers support `logger` and
`clock`; their requirements silently propagate to callers. Native temporal
provider bindings and borrowed access to a caller's output remain pending.

Parameter-level `const` retains its existing meaning: fixed wiring-time
configuration on a temporal callable. Function-level `const` is not shorthand
for adding that qualifier to every parameter. A source configured entirely by
fixed values can still tick, so an all-`const` signature is not evidence that
the function is non-temporal.

### HGL example and C++ expectation

The following uses `const fn`:

```hgl
const fn scale(value: f64, factor: f64) -> f64 =>
    value * factor

fn scaled(value: f64, const factor: f64) -> f64 {
    when modified(value) && valid(value) {
        return scale(value, factor)
    }
}
```

The expected C++ shape is a plain value helper called by the containing node's
evaluation hook, not another node. Illustrative lowering, not current emitter
output:

[Native example 1](https://github.com/hhenson/hgraph_spec_audit/blob/main/examples/documentation/language/docs/design/decisions/0008-temporal-contracts-and-target-mappings.md#example-1)

For this single-input example, the node's activation/readiness policy supplies
the handler's modified/valid admission. `scale` neither schedules evaluation
nor emits the result; `scaled` does. A permitted call to `scale` on two
wiring-time scalar values instead computes a scalar immediately, without
wiring a node. A temporal port cannot be passed to that helper as a scalar in
graph composition: there is no current payload to read at wiring time. Such
a call instead uses the default lifting rule below.

### Role selection and default lifting

In graph composition, a declared temporal `fn` takes precedence over a
same-named `const fn`. Resolve its signature normally; a type error is not a
fallback trigger. Only when there is no temporal definition does the normal
call select the value definition. Node evaluation, lifecycle hooks, and value
function bodies instead select the value role exclusively.

`const(function)` is an explicit value-role selector. For example,
`const(scale)(value, factor)` bypasses a same-named temporal definition, and
`eval(const(scale), value: [1.0, 2.0], factor: 3.0)` tests that value definition
through the shared lifting mechanism. This wrapper selects; it does not
itself invoke, lift, promise purity, or change native phase permissions.

A value call with temporal arguments in graph composition becomes one runtime
node equivalent to `when { return value_function(...) }`. Temporal arguments
are inputs; scalar arguments and omitted defaults are configuration. The
default is any input modified and all inputs valid, using ordinary endpoint
validity, without the immediate-child checks of `all_valid`. Outputless functions become sinks. An
all-scalar call executes directly, with no invented source timing. Custom
activation/validity belongs in an explicit temporal wrapper.

`eval` follows temporal-first selection unless `const(function)` overrides it.
For a selected value function, sequence literals identify the driven inputs
and scalar arguments identify configuration. At least one driven input is
required. The harness does not implement a second scheduling policy.

### Compiler layering

An implementation could resolve execution roles and lifting before native
emission. The checked call must retain argument binding, capabilities and
result type; the emitter must not reselect a different contract.


### Operators and native implementations

Execution role and implementation language are independent. One nominal
operator may have temporal and value-level candidates, and either role may
have HGL or native implementations. A native graph implementation wires a
graph; a native value implementation directly operates on its admitted
arguments. Being native does not determine the role.

The call context first constrains candidate eligibility. Selection must retain
the operator identity, concrete type domain, signature, and applicable
constraints; it must not depend on a coincidentally equal name or rediscover
an overload on each tick. The existing hgraph resolver remains the owner of
temporal candidate matching/ranking. Extending the model to value candidates
must preserve its shared matching rules, not add an independent dispatcher.

The [fixed symbol-to-name mapping](../operators.md#fixed-symbol-to-name-mapping)
is unchanged. For example, `*` identifies `mul_`; the context and domain
determine an eligible implementation. Domain-bound algebraic properties do
not automatically transfer to a different numerical policy or candidate.

A value `const fn` can supply the default lifted node described above. This
does not automatically register a new implementation of an unrelated nominal
operator or define an `impl const fn` syntax. Conversely, having a temporal
operator does not make it callable as scalar work inside a node.

Native declarations now follow [ADR 0014](0014-native-implementation-interfaces.md):
`native fn` is temporal and `native const fn` is value-level. Operator
implementation and export modifier combinations remain separate work. Scalar
helpers now use `native const fn`; legacy inline view helpers remain until
the explicit collection-borrow contract is settled.

## Cache versus recordable state

The language distinguishes semantic history from reconstructible local data:

| HGL concept | Meaning | Current native counterpart | Record/replay |
| --- | --- | --- | --- |
| `state` | Persistent data needed to determine subsequent computation | `RecordableState<TSchema>` | Recorded and restored |
| `cache<T>` | Node-local data reconstructible independently of missing history | `State<T>` | Excluded; rebuilt after restart |

`cache<T>` names the agreed concept. The scalar declaration and initializer grammar is settled in
[ADR 0011](0011-cache-declarations.md). It does not rename C++ `State` or introduce
a native `Cache` selector.

Given the same restored inputs and recordable state, restarting with an empty
or reconstructed cache must preserve subsequent output values, validity,
ticks, deltas, and semantic side effects. Only the cost of producing them may
differ. Incremental cache maintenance is fine if the contents can also be
recovered from authoritative current data without replaying lost history just
to rebuild the cache. Rebuilding must finish before any evaluation that
depends on the contents; inputs need not be available during physical
construction, so a cache may initially be empty and explicitly not ready.

Examples:

- An index derived from the complete current input map is a cache if it can
  be rebuilt after restoration without changing observable results.
- A cached REF is suitable when current input connections and a current or
  restored selection identify its source. A historical selection known only
  to the cache is semantic state and must instead be recordable.
- Pending native alarms keep their authoritative deadlines and tags in a
  dedicated scheduler checkpoint; finite operator progress belongs in recordable
  state. Scheduling indices are rebuilt after restore
  ([recovery contract](0011-cache-declarations.md#scheduler-recovery-contract)).
- A running total, last-seen value, or queue of unconsumed events is not a
  cache merely because it is stored privately. When missing history is needed
  to reconstruct it, use recordable state or an explicit temporal structure.

REF opacity and binding-change semantics remain unchanged. Caching a reference
does not grant payload access below it, serialize an engine handle, extend a
borrowed view's lifetime, or keep a handle valid after its owning graph dies.

### Construction and typing

Cache and state storage are both planned and their objects constructed during
node initialization, before `start`. There is no physical lifetime distinction
that lets cache allocation or construction be deferred to `start`. Constructing
the object is separate from populating its logical contents or restoring
recorded values. Replay-aware state initializers must not overwrite restored
state; cache reconstruction must use the restored authoritative data.

Normal teardown runs semantic `stop` before destroying the constructed
objects and releasing their storage. Partial initialization must clean up
whatever was successfully constructed without assuming `start` completed.
Native static nodes destroy constructed slots after a failed `start`, without
calling that node's `stop`; partial resource acquisition must use RAII or a
rollback guard. Generic HGL native construction hooks remain future work.

Cache follows state's applicable typing and lifetime rules but does not
require recordability. It may therefore contain admitted native/non-recordable
types. HGL `state` still requires recordable types; a native type is eligible
there only if its recordability contract is supplied. Opaque native storage
does not remove this distinction. Cache is also not a blanket permission to
own external resources or introduce I/O outside a native lifecycle contract.

A node may declare both recordable history and a derived cache. Restoration
precedes `start`; cache rebuilding uses the restored state. Their storage may
be planned independently, but that choice must not change initialization order
or observable recovery behavior.

For **HGL-MIG-005**, this settles the reconstructible-cache distinction, not
generic recordable-state construction. Non-default-constructible generic
state, sparse validity, queues, and windows still require their own accepted
representation and initialization rules. They must not be relabelled as
cache to bypass record/replay.

## Language contracts and target mappings

Keep four concerns distinguishable, without prescribing four new source
declaration forms:

| Concern | Information it owns |
| --- | --- |
| Semantic contracts | Nominal type and callable identities, signatures, behavior, execution roles, ownership/effects, and required capabilities |
| Requirements and use | Which contracts and capabilities a module needs, without naming a provider's implementation spelling |
| Implementations | Candidates satisfying those contracts, their domains/constraints, and HGL or native bodies |
| Target mappings | Concrete representations, lifecycle operations, symbols/wrappers, engine APIs, ABI, and build/load dependencies |

A target is more specific than an emitted language: it includes the engine,
language, and relevant ABI/platform/profile. Two C++ engines may share scalar
representations and numerical helpers while using different node/view APIs.
A Rust or Zig realization must preserve the same semantics without pretending
those C++ types or ownership operations exist there. Mapping reuse and
overrides need explicit, deterministic compatibility rules; their declaration
syntax and composition mechanism are still open.

Stable language identity is distinct from a target layout or ABI fingerprint.
Where a type already exists in hgraph, reuse its canonical identity and
operations rather than registering a duplicate identity from its HGL name.
An absent or incompatible mapping/capability must fail during compilation or
planning, not silently substitute different behavior.

### Native type realization and lifecycle

The discussion's illustrative sketch was:

[Native example 2](https://github.com/hhenson/hgraph_spec_audit/blob/main/examples/documentation/language/docs/design/decisions/0008-temporal-contracts-and-target-mappings.md#example-2)

This is a design sketch, not accepted parser syntax. `SomeType` would be an
atomic nominal language type; `some::Type` would be a target mapping, not the
type's portable meaning. A colocated mapping could be convenient, but the
model must also permit using the contract with another target's mapping.
Native fields and layout do not become visible automatically.

A type realization needs an explicit lifecycle protocol covering:

- memory requirements, including size and alignment for a concrete realization;
- construction in supplied storage and destruction of a live object;
- supported copying, ownership transfer/move, borrowing, and sharing;
- who allocates and releases storage, separately from who constructs and
  destroys an object;
- optional capabilities such as equality, hashing, ordering, and recordability,
  required only by contexts that use them.

These are semantic capabilities, not a requirement to imitate every C++
special member on every target. Destroying an arena-owned object must not free
the containing arena. Retaining/releasing a Python-backed reference is not
automatically an independent or deep value copy. Unsupported operations must
remain unavailable rather than acquire an accidental byte-copy fallback.

Native lifecycle helpers implement this protocol; an ordinary function merely
named `init` does not become a constructor automatically. Association with the
type, receiver/result ownership, supported arguments, allowed phases, failure
handling, and cleanup need explicit contracts. Their source spelling remains
open. Node `start`/`stop` hooks and type construction/destruction are separate
layers, even when both ultimately call native functions.

## Target boundary

A target binding preserves the language contract: execution role, type,
ownership, effects, capabilities and lifetime. It may choose native types,
symbols and storage without changing those obligations. A new target requires
independent validation of the same acceptance scenarios.

Acceptance covers call phase, lifting, operator selection, construction before
`start`, cache rebuilding after restore, cleanup on abort, borrowing and REF
lifetimes, and missing-capability diagnostics.
