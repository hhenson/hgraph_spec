# Generic struct shape arguments

A struct type argument must satisfy every occurrence of its parameter:

- An ordinary value position requires canonical `value_type`, including T in
  `value: T`, `list<T>`, and the payload of `atomic<T>`.
- Inside `delta<S>`, parameters may describe temporal shapes, provided the
  substituted S has an admitted ordinary delta type.
- Passing a parameter to another generic application propagates that
  declaration's parameter requirements. Apply this through fields and parents;
  intersect repeated uses and the existing `requires` constraints.

Check these obligations symbolically and after substitution. An unresolved,
conflicting or unsupported application is a checking error; it does not create
a partial specialization. An unused parameter gains no temporal-shape permission
from this rule. No new constraint predicate, grammar or runtime type test is added.

```hgl
struct Publication<T> { value: delta<T> }
struct Batch<T> { values: list<Publication<T>> }
struct Mixed<T> {
    value: T
    publication: delta<T>
}
```

`Batch<T>` forwards Publication's delta-formation requirement. `Mixed<T>`
also requires T to be an ordinary value type; the delta occurrence does not
relax that restriction. Concrete shape admission remains the
[ordinary delta contract](ordinary-delta-types.md), not a property of a
particular struct name.

Specializations retain their complete invariant source arguments, including
nominal arguments, fixed sizes and temporal boundaries. Equal derived payload
types do not merge distinct source arguments. Fields contain the resulting
ordinary values; allowing a temporal shape as an argument does not store an
endpoint. Constructor inference still requires a unique complete substitution
under the existing delta-matching rules.

[Compiler cases](../../../compiler/cases_struct_shape_arguments.md) cover
forwarding, intersections and rejection.
