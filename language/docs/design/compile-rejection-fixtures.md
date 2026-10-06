# Compile-rejection fixtures

`hgl test --reject fixture.hgl` checks a source file against its expected
source errors. It does not execute tests or run a graph. A fixture must
contain at least one expectation and must fail source checking. Success exits
zero; a reported fixture failure exits 1. Infrastructure failures exit nonzero
and must be identified as infrastructure failures, never as a matched rejection.
Ordinary `hgl test` does not interpret these comments as permission to reject
source. Keep intentional rejection fixtures separate from successful examples.

Only a real standalone HGL line comment is an annotation. Text inside string
literals or block comments is not metadata; this follows lexical context even
in a fixture with invalid source. This adds no string or comment syntax.
An expectation occupies a whole comment line:

```hgl
# expect-error(type, "rolling.size_kind")
fn consume(value: rolling<f64, 5m, 3>) { when { } }
```

The category is an unquoted name from the [diagnostic catalogue](error-catalogue.md#diagnostic-categories);
the code is one string literal from its [source-error table](error-catalogue.md#source-errors).
Hyphenated category names such as `function-kind` are single annotation names.
This annotation vocabulary does not introduce bare identifier values into HGL
expressions. Leading/trailing whitespace is allowed; the annotation has exactly
two arguments and no trailing comment. Unknown categories, unknown codes,
category/code mismatches and malformed `# expect-error` annotations fail
fixture validation before matching. `build` is recognized but not eligible:
toolchain or infrastructure failure cannot prove source rejection.

Match an error's primary location to the fixture's exact source file and the
physical line immediately after the annotation. Do not skip blank or comment
lines. Matching uses the first character of the primary source range; its
column, range end and message text are not compared. An EOF error uses the
line containing the EOF position, so it is matchable only when that is the
annotated line. An annotation without a following source line is invalid.

Every annotation must match exactly one primary error with the exact category
and code, and every primary error must be matched exactly once. Missing,
duplicate, uncoded and unexpected errors fail. Errors in another source file
cannot satisfy an expectation in this file. Attached notes and related
locations do not count as additional errors; warnings are reported but do not
participate in matching. Error order does not affect the result.

Each annotated line should isolate one rejection: do not put several invalid
constructs on it. A fixture may have several annotations on different lines,
subject to ordinary diagnostic recovery. Do not require recovery to invent
missing diagnostics. Successful source checking, a later build failure, crash
or timeout always fails the fixture, regardless of expectation text.

This initial command accepts one fixture path and no test-name selectors or
module-part options. Module dependencies follow ordinary resolution. It must
finish source diagnostics before any build or execution step.

See the deliberately separate [rejection fixtures](../../examples/reject/README.md).
