# Execution-error assertions

Inside a named test, `assert raises("code") { ... }` requires execution of
the block to raise the specified [execution error](error-catalogue.md).
This includes a catalogued eval input-admission failure before graph execution.
The argument must be one string literal naming a listed code. Unknown codes
and computed arguments are source errors. `raises` is contextual immediately
after `assert`; it is neither a function nor a general exception construct.

```ebnf
assert_statement = "assert", ( expression | raises_assertion );
raises_assertion = "raises", "(", expression, ")", block;
```

The block is a lexical test-body scope. It admits the same bindings,
expressions, calls, composition-phase conditionals and iteration, and assertions
as its enclosing test, including nested `assert raises`. Existing phase rules
apply: it cannot declare functions or tests, inject capabilities, contain
runtime handlers or `yield`, or return from the test. Locals do not escape.
An `eval` may be used as a statement even when its result is discarded.

Execute the block exactly once, in source order. An execution error escaping
it stops the remaining statements. Match the error's structured code exactly,
never its message, category, native exception type, or a substring. A matching
error passes the assertion only after ordinary graph teardown and block cleanup
complete successfully; continue at the statement after the assertion. The
block's normal completion, a different code or an error without a listed code
fails the test. A failed test does not execute subsequent test statements. The test runner
reports the failed test and exits 1 when any executed test fails.

Assertion failures, including failed nested `raises`, are test failures and
cannot satisfy any enclosing `raises`. Source-checking and build failures,
crashes, timeouts and cleanup failures also fail; none becomes an expected
execution error. An error raised only during cleanup cannot satisfy `raises`.
If cleanup fails after a matching execution error, report both failures and
fail the test. Successful inner assertions consume their errors, so an outer
block must itself raise its expected error to pass.

Existing graph error propagation and effects are unchanged. Earlier completed
publications, state writes and external effects remain; no rollback or retry
is introduced. Each `eval` retains its ordinary run ownership and teardown.
If the graph captures an error and publishes it as data, it has not raised
that error to this assertion: inspect its output with ordinary assertions.
This adds no error type, catch binding, rethrow, or production exception syntax.

See [executable examples](../../examples/execution-errors.hgl).
