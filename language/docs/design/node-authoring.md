# Nodes and native bindings

Status: native binding contract.

Author library behaviour in HGL. Guards, state, scheduling and writes remain
in the language; selected native helpers supply target-specific operations.

A temporal operator can call a scalar native helper:

```hgl
impl fn bit_and(lhs: i64, rhs: i64) -> i64 {
    when { return native::bit_and(lhs, rhs) }
}
```

The node owns temporal inputs, activation and publication. The helper
receives two values; it cannot schedule, publish or retain an input view.

## Binding contract

- **NAT-1:** Select by canonical module/declaration identity, full signature and
  role. `native const fn hgraph.native.bit_and(i64,i64)->i64` is a value helper;
  `hgraph.operators.bit_and` is a temporal operator. Neither substitutes for
  the other. Resolve candidates in the checker, retain the selection in IR.
- **NAT-2:** Generate the implementation interface from the HGL contract. C++
  uses a checked `bind<T>()` adapter; Rust implements a generated trait. Keep
  implementations in native source and dependencies in its normal library build.
  Select the provider once, without a separately authored function-symbol map.
  Missing, ambiguous or incompatible providers are compile errors.
- **NAT-3:** Value helpers receive admitted payloads. Input-view helpers borrow
  the current input, including its local binding state. The borrow ends with
  the call; no copying collections, retaining views or resolving output names
  per tick. `signal` queries must also work before validity.
- **NAT-4:** A temporal implementation owns activation, guards, state, lifecycle
  and writes. A helper call adds no node and emits no tick on its own. Keep
  equal ordinary ticks; suppress equal REF designations under TS-16.
- **NAT-5:** HGL `state` is semantic history, requiring checkpoint restoration;
  `cache` must be reconstructible. Native storage alone does not establish
  record/replay behavior. Fallible helpers preserve the declared error policy
  at the node boundary; host-language panics are not that policy.
- **NAT-6:** Validate a shaped handle at construction. Metadata helpers inspect
  the input view for peered and assembled TSL/TSB, TSD and REF. Do not infer
  input time or validity by inspecting only its peer. Preserve the accepted
  removal, invalidation, rebinding and child-scope rules.
- **NAT-7:** Check capability requirements for HGL and native functions alike.
  Calls silently add the callee's requirements to the caller, transitively and
  without duplicates. `inject out` refers to the declared temporal result. Value functions may use
  admitted context services without acquiring a node. Selected HGL implementation parts declare
  target requests and lifecycle hooks; the shared declaration owns the signature. See the
  [capability contract](native-interfaces.md#implementation-parts-and-injectables).

`ref<ref<T>>` in source is an error. Substituting `T = ref<U>` into `ref<T>`
normalizes to `ref<U>` before target mapping. No nested REF runtime endpoint.

`native fn` follows ordinary temporal typing. Only `native const fn` is a
value helper; parameter-level `const` does not change execution role. Borrowed
input access must be explicit in the shared contract.

## Acceptance

Check wrong signatures, forbidden phases, escaping borrows, ambiguous
candidates and missing capabilities. Binding checks and temporal trace
conformance are separate obligations.


## Compiler acceptance example

The [const/debug graph](../../examples/const-debug.hgl) tests source scheduling,
admission, publication and composition. Its expected ticks and effects are
independent of the chosen native implementation.
