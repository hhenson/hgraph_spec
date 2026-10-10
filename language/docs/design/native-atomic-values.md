# Native atomic values

A native atomic value is an opaque nominal scalar, supplied by a checked
provider. It is ordinary data, distinct from native resource state.

## Rules

- **NVAL-1 — Declaration.** `native type Token` declares a module-local opaque
  nominal value binding; `export native type Token` exposes it. The package
  descriptor maps that qualified declaration to a canonical native scalar
  identity under ADR 0008. Reuse an existing identity; do not duplicate it
  under the HGL name. Two bindings mapped to the same identity are aliases.
  No fields, inheritance, type arguments, representation or implicit conversions are exposed. Import the type through
  ordinary module `use`. Equal layouts or text do not equate different
  canonical types.
- **NVAL-2 — Provider contract.** Bind each declaration to its canonical scalar
  metadata before execution. The existing package descriptor records the shared
  type contract: owning copy and text operations, optional equality/hash/order, and any serialization
  required by an admitted context. Every target satisfies that same contract
  rather than independently choosing capabilities. Checking reads it without
  loading code; native class definitions do not infer capabilities. Missing or incompatible providers fail before
  execution. Retained values keep their provider available through destruction.
- **NVAL-3 — Construction and access.** Checked `native const fn` helpers may
  construct, return or read the exact ordinary type in their permitted phases.
  There is no implicit type-name constructor or structural access. Consumer
  arguments are read-only; retained results own independent data under
  VAL-16/17. A typed global-state read copies a native atomic value; replacing
  its local `var` does not write the entry. Rebinding a `var` replaces its
  complete value. This adds no native
  mutation helper, borrow escape, resource-state conversion or arbitrary
  compile-time execution. Executed test setup and cold materialization may
  call a helper explicitly permitted there, without runtime-only capabilities; this is ordinary
  execution, never native code run by the source checker. Defaults and key
  recipes retain their existing required-constant/cold-phase restrictions.
- **NVAL-4 — Capabilities.** Source operations require the provider's declared
  capabilities. Missing capabilities fail checking where the type is known.
  Boxed use follows ANY-3/5 and `value.capability`. Supported equality/hash
  preserve canonical identity and content; order may be partial. Opaque
  representation is never compared bytewise or by address. Serialization is
  required only where the enclosing context requires recordable values.
- **NVAL-5 — Publications.** Admit native atomic leaves recursively in sparse
  structures, complete ordinary payloads, boxes and rolling arrivals.
  `atomic<Token>` is `Token`; `delta<Token>` is `Token`. Generic pass-through,
  eval, replay, record and `TimedValue<T>` retain complete independent values.
  Equal publications tick; `_` is silence. Set/map key use requires equality
  and hash and the existing cold key rules. Distinct canonical native identities
  remain distinct even when boxed. Opaque resource state remains inadmissible
  on temporal ports.

[Examples](../../examples/native-atomic-values.hgl) specify provider-independent
HGL tests; each backend supplies its own implementations of the same native
interface. [Acceptance cases](../../../compiler/cases_native_atomic_values.md)
include provider and capability failures.
