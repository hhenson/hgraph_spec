# Compile-rejection fixtures

These files are intentionally invalid HGL. Exclude this directory from
successful-example discovery. Run each file separately, for example:

```sh
hgl test --reject language/examples/reject/rolling-size-kind.hgl
```

The [fixture contract](../../docs/design/compile-rejection-fixtures.md) requires
an exact category, code, file and next-line match for every primary error.
The [catalogue](../../docs/design/error-catalogue.md) enumerates allowed names.

- `missing-parameter-close.hgl`: a missing required `)` token.
- `rolling-size-kind.hgl`: different maximum/minimum size kinds.
- `rolling-size-bounds.hgl`: a minimum exceeding its maximum.
- `yield-time-type.hgl`: an integer where a yield time is required.
- `raises-unknown-code.hgl`: a literal that is not a known execution code.
- `raises-computed-code.hgl`: a computed argument rather than a string literal.
- `test-inject.hgl`: a runtime capability declaration inside a test.
- `comment-text.hgl`: expectation-like text in strings and block comments is ignored.
