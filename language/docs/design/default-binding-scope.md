# Default expression binding scope

Defaults are declaration metadata. A default does not compute from the
arguments supplied to that invocation or the fields of the instance being
constructed.

- **DEF-1 — Binding closure.** Resolve field defaults, inherited default
  overrides and callable parameter defaults in their declaration's enclosing
  lexical/module binding closure, with its existing generic environment.
  The declaring struct's instance-field names and the declaring callable's
  formal-parameter bindings are not in this default-expression scope.
  Imported declarations keep their defining binding closure.
- **DEF-2 — Arguments are unavailable.** Earlier, current and later formal
  argument values are unavailable to a default, including parameter-level
  `const` configuration. An instance field likewise supplies no default
  value. If the same spelling names an admitted enclosing declaration, the
  default resolves that declaration; the function body still follows ordinary
  parameter shadowing. Otherwise ordinary name checking rejects the reference.
- **DEF-3 — Generic context.** Preserve generic type and size relationships
  in a default's expected type and expression. Apply the existing complete
  substitution and specialization checks before execution. Substitution keeps
  the defining binding closure; it does not capture caller declarations or
  grant numeric generic reification. Generic-parameter defaults remain excluded.
- **DEF-4 — Existing constant boundary.** Defaults still require expressions
  admitted by their existing required-constant/cold context and canonical
  expected type. The expression grammar alone grants no additional operation,
  phase or capture. Ordinary `const fn` value parameters may have defaults;
  ordinary temporal parameters may not. Default ownership and retention retain
  their existing contracts.

```hgl
const fn seed() -> i64 => 7
const fn choose(seed: i64, value: i64 = seed()) -> i64 => seed + value
```

`choose(1)` returns 8: its default calls the enclosing `seed` declaration,
while its body reads the supplied parameter. A default written `value = prior`
where `prior` names only another parameter is rejected, whether that parameter
is ordinary or explicitly `const`.

`struct Defaults<T> { items: list<T> = [] }` supplies an existing contextual
empty-list default checked with the concrete specialization's `list<T>` type;
it neither reads an instance field nor reifies T as a value.

See [source cases](../../../compiler/cases_default_binding_scope.md),
[name resolution and specialization](../developer-guide/syntax-and-semantics.md),
[value-function defaults](../user-guide/value-functions.md),
[struct default retention](struct-constructor-order.md), and
[module binding closures](modules.md#cross-module-retained-specialization).
