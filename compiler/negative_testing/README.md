# Negative-testing conformance

The [case manifest](cases.json) uses schema version 2. Every case invokes ordinary
`hgl test FILE`, followed by any `select` names and one `--part FILE` per `parts`
entry. All paths are relative to this directory. Every case is independent and
bounded; `controls/` intentionally contains failing cases and must not be
discovered as a passing-example suite.

Each entry has a unique `id`, `file`, `mode` (always `test`), `expected_exit`
(0 for success, 1 for reported failure), `purpose`, and `preflight` flag.
Optional fields give additional required observations:

- `runtime_results`: named tests that must execute, with `outcome` of `passed`
  or `failed`.
- `rejection_results`: checked rejection cases with the same outcomes, identified
  by test `name` or by original `file` and declaration-start `owner_line`.
- `not_executed`: named tests that must not execute.
- `select` and `parts`: test-name selectors and explicit module-part paths.

Observe each result in the command's report; exit status alone does not prove
execution or rejection. The report's text format is not prescribed. A source
checking failure does not satisfy an expected runtime failure. Crashes, signals
and timeouts never satisfy expected control outcomes. `preflight: true` marks
pure runtime fixtures that can also pass ordinary source checking; false means
source preflight must not be used as an admission requirement for the case.

The [execution-error contract](../../language/docs/design/execution-error-assertions.md)
and [source-rejection contract](../../language/docs/design/compile-rejection-fixtures.md)
define the outcomes; the [catalogue](../../language/docs/design/error-catalogue.md)
defines accepted codes.

The mixed cases check continuation after a rejection mismatch, whole-test
exclusion, nested test contexts, selectors and module parts. The original
`missing-parameter-close` case retains its missing delimiter: its sole
declaration has a reliable extent through EOF. `syntax-recovery` has balanced
delimiters and a missing required token, followed by a test that must execute.
`unsafe-recovery` has unclosed nested bodies before a neighbour and must fail
admission without executing that neighbour. These are distinct boundary cases,
not a blanket prohibition on missing delimiters.

Execution controls cover normal completion, empty blocks, wrong codes,
assertion failures and consumed inner errors. Source controls cover exact
locations, extra errors and invalid metadata. Cleanup failures, crashes and
timeouts must also fail under the contract; these portable bounded fixtures do
not induce process or cleanup failures.
