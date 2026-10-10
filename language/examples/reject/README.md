# Source-rejection examples

These examples contain annotated source errors. Run them with ordinary test
commands, which check each rejection case and execute surviving tests:

```sh
hgl test language/examples/reject/rolling-size-kind.hgl
hgl test language/examples/reject/mixed.hgl
hgl test language/examples/reject/mixed.hgl rejected_test
hgl test language/examples/reject/mixed-parts/main.hgl --part language/examples/reject/mixed-parts/helpers.hgl --part language/examples/reject/mixed-parts/cases.hgl
```

The [contract](../../docs/design/compile-rejection-fixtures.md) requires an exact
category, code, file and next-line match for every primary error in each isolated
case. The [catalogue](../../docs/design/error-catalogue.md) enumerates allowed
names. Ordinary source-checking commands still reject these annotated errors.

- `missing-parameter-close.hgl`: a missing required `)` in a sole declaration.
- `missing-declaration-name.hgl`: an isolated declaration without a valid name.
- `invalid-import.hgl`: a rejected `use` declaration.
- `syntax-recovery.hgl`: a missing token in balanced syntax, with a surviving test.
- `rolling-size-kind.hgl`: different maximum/minimum size kinds.
- `rolling-size-bounds.hgl`: a minimum exceeding its maximum.
- `yield-time-type.hgl`: an integer where a yield time is required.
- `raises-unknown-code.hgl`: a literal that is not a known execution code.
- `raises-computed-code.hgl`: a computed argument rather than a string literal.
- `test-inject.hgl`: a runtime capability declaration inside a test.
- `comment-text.hgl`: expectation-like text in strings and block comments is ignored.
- `mixed.hgl`: declaration and whole-test rejection alongside executable tests.
- `nested-context.hgl`: inner owners leave sibling tests and helpers available.
- `mixed-parts/`: named rejection tests and surviving helpers across explicit parts.

- `delta-errors.hgl`: unsupported delta formation, reduced scalar/enum/atomic/
  rolling payload errors, exact structural identity, nested child types,
  constructor names/constants/types/duplicates/bounds/overlap, and valid controls.
  State-dependent trace failures instead use `assert raises` in
  [`eval-profile-errors.hgl`](../eval-profile-errors.hgl).

- [`default-binding-scope.hgl`](default-binding-scope.hgl) is a standalone
  negative checking input for earlier/current/later parameter values and
  instance-field references in defaults. These failures remain uncoded; it is
  not an in-source coded-rejection fixture.
