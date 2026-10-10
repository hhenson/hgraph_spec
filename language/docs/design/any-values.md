# Any values and publications

`any` is an ordinary owning box containing a typed value or nil. Its contents
may change type; the box's canonical type remains `any`.

## Rules

- **ANY-1 — Construction.** `any()` constructs an empty box. `any(value)`
  takes one admitted ordinary value, evaluates it once and independently
  retains its concrete type and contents. Passing an `any` copies its contents
  without another box layer. There are no named arguments or implicit boxing
  conversions. Readable ordinary payload observations may be consumed by
  retaining their data, including `any(delta_value(input))`. No endpoint or borrowed view
  handle is stored. Temporal connections at wiring time and opaque resource
  state cannot be boxed. Construction follows the call's ordinary phase and
  adds no implicit temporal lift.
- **ANY-2 — Ownership.** Apply VAL-16/17 recursively to construction and
  retention. Source mutation, replacement or teardown cannot change a retained
  box. Replacing a `var` with another `any` may change its contained type but
  never the binding's type. A box preserves a concrete family member's identity.
  `get(global_state, key)` for `any` follows aggregate lexical borrowing:
  `let` borrows read-only; `var` borrows exclusive writable entry access.
  `any(observed)` independently retains the contents; the borrow cannot escape.
- **ANY-3 — Operations.** Two empty boxes are equal. Empty precedes populated.
  Populated boxes of different contained canonical types are unequal and
  unordered; no numeric coercion or type-name ordering applies. For the same
  contained type, equality, hash and order delegate to that value. Equal boxes
  have equal hashes, including empty boxes, which are admitted keys. Unordered
  comparisons make each of `<`, `<=`, `>` and `>=` false. An executed operation
  requiring a missing contained capability fails with `value.capability`;
  a required constant with a statically known missing capability fails checking
  with category `type` and code `value.constant_capability`.
  Optional folding preserves execution failures. Earlier effects stand.
  These rules implement VAL-11; they add no unboxing, casts, type inspection or
  prescribed text/hash representation.
- **ANY-4 — Publications.** `any` is one temporal leaf, irrespective of the
  contained shape. `atomic<any>` normalizes to `any`; `delta<any>` is `any`.
  Admit it recursively in sparse structures, complete atomic payloads and
  rolling arrivals. Contained values use the existing admitted ordinary kinds,
  including finite recursive, family and structural delta values. Boxing a delta retains its
  originating type and entries; it does not apply them to a hidden endpoint.
  A publication replaces the whole box. Empty and equal boxes tick; `_` denotes silence.
  Replay, record and `TimedValue<T>` retain complete boxes through the existing
  generic pass-through. Harness comparison follows ANY-3, including capability
  failure rather than silently declaring a mismatch.
- **ANY-5 — Keys.** Admit `any` set members and map keys where ordinary key
  recipes are admitted. Key use requires the contained value's equality and
  hash; missing capabilities fail with `value.capability` during executed
  construction or cold materialization. Required constant evaluation rejects
  them with category `type` and code `value.constant_capability` when the missing
  contained capability is statically known. Existing non-NaN key restrictions
  apply recursively through the box;
  this extension defines no NaN key identity. Boxed different types remain
  distinct keys. Duplicate and overlap checks retain their existing phase and exact value rules.

[Examples](../../examples/any-values.hgl) and
[required cases](../../../compiler/cases_any_values.md) define acceptance.
Native atomic construction is a separate provider contract; REF designation
is not an ordinary boxed payload.
