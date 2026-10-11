# Atomic scalar equivalence cases

These check [atomic scalar equivalence](../language/docs/design/atomic-scalar-equivalence.md).
They specify compiler behavior; they are not measured implementation results.

| Case | Required result |
|---|---|
| Compare S with `atomic<S>` for every non-composite value type, including all thirteen built-in scalars and enums. | Equal canonical types; no conversion or second endpoint schema. |
| Compare `atomic<Mode>` with `i64` or another enum. | Unequal; retain Mode's nominal identity. |
| Compare `list<atomic<i64>, 2>` with `list<i64, 2>`. | Equal, recursively normalized child types; preserve fixed size. |
| Compare `TimedValue<atomic<i64>>` with `TimedValue<i64>`. | One nominal specialization, with the same scalar delta field. |
| Apply `Box<T> { value: T }` to `atomic<i64>`. | Normalize the generic argument to i64 before checking its ordinary-value occurrence. |
| Match an admitted scalar S against `delta<T>`, with or without an explicit atomic spelling. | Bind T to canonical S; no additional inference alternative. |
| Declare overloads distinguished only by S versus `atomic<S>`. | They do not become distinct types; apply the existing duplicate or overlap rules. |
| Use `const value: atomic<i64>`. | Reject: the annotation grammar still requires `value_type`. |
| Compare a composite V with `atomic<V>`, including a single-field nominal struct. | Distinct temporal shapes; do not erase the atomic boundary. |
| Form a delta for a scalar outside the admitted publication profile. | Equivalence does not extend delta admission. |
