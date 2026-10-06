# Negative-testing conformance

The [case manifest](cases.json) uses unique `id` values, `file` paths relative to this directory,
`mode` (`test` or `reject`), `expected_exit` (0 for success, 1 for failure),
and a short `purpose`. Invoke `hgl test FILE` for `test` and
`hgl test --reject FILE` for `reject`. An exit of 1 denotes the command's
reported test or fixture failure. For runtime controls, confirm that the named
test executed and failed its assertion; a source-checking failure is not the
expected outcome. Crashes, signals and timeouts do not count
as expected control outcomes. Every case is independent and bounded.

The [execution-error contract](../../language/docs/design/execution-error-assertions.md)
and [compile-rejection contract](../../language/docs/design/compile-rejection-fixtures.md)
define the outcomes; the [catalogue](../../language/docs/design/error-catalogue.md)
defines accepted codes. `controls/` intentionally contains failing tests and
invalid expectations. Never discover it as a passing-example suite.

The positive execution example checks both yield errors, successful nesting,
and fresh evaluations after expected failure. Controls prevent false passes
for normal completion, empty blocks, wrong codes, assertion failures, consumed
inner errors, wrong source locations, unexpected errors and invalid metadata.
Cleanup failures, crashes and timeouts must also fail under the contract;
these portable bounded fixtures do not induce process or cleanup failures.
