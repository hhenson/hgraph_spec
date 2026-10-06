# Source-rejection tests

`hgl test file.hgl` runs executable tests and checks expected source errors in
that file and its supplied module parts. A standalone `# expect-error` comment
marks a rejection case; there is no separate rejection command or flag.
A file may contain either kind of case or both.

```hgl
module examples.mixed_tests

# expect-error(type, "rolling.size_kind")
fn invalid(value: rolling<f64, 5m, 3>) { when { } }

test invalid_test {
    # expect-error(phase, "test.statement_phase")
    inject clock
    assert false
}

test still_runs { assert true }
```

Here the invalid function and the whole `invalid_test` are checked for
rejection. No statement in `invalid_test` executes, including statements before
its annotation. `still_runs` executes normally.

## Annotation and ownership

Only an actual standalone HGL line comment is metadata. Text in strings or
block comments is ordinary source text, including during syntax recovery.
The annotation is exactly `# expect-error(category, "code")`, with optional
surrounding whitespace and exactly two arguments; no trailing comment is
allowed. The category is an unquoted name from the
[diagnostic catalogue](error-catalogue.md#diagnostic-categories), including
hyphenated names such as `function-kind`. The code is one literal from its
[source-error table](error-catalogue.md#source-errors). These names do not
introduce bare identifier values into HGL expressions.

The target is the physical line immediately after the annotation; never skip
blank or comment lines. Its owner is the enclosing named test, or otherwise
the enclosing declaration. An annotation immediately before a declaration
belongs to that declaration when its target line begins it. Inside an unnamed
`test { ... }` context, the inner helper declaration or named test owns the
annotation, not the context. A `use` declaration can own a case. A malformed
declaration need not have a recoverable name, but must have reliable boundaries.
Module headers and unnamed test-context headers cannot own cases. An orphaned
annotation, or one without a following source line, is invalid metadata.
Multiple annotations with the same owner form one rejection case.

Unknown categories or codes, mismatched category/code pairs and malformed
annotations fail admission. `build` is recognized but ineligible: infrastructure
failure cannot prove source rejection. Metadata applies to the explicit target
module and supplied parts; imported dependencies retain ordinary checking.
Their comments do not suppress source errors in those dependencies.

## Isolation and checking

Determine every case's ownership and extent before execution. Syntax recovery
must preserve the surrounding declarations and tests. If a missing delimiter
or other damage makes a boundary ambiguous, fail admission; do not consume a
neighbouring test or choose a boundary merely because a new line or declaration
keyword was encountered. Recoverable missing tokens may still be expected
errors, with their ordinary diagnostic locations.

Remove every rejection owner from the executable module, whether selected or
not. Check the remaining source normally, including its imports and parts.
Check each selected rejection case independently in the same module scope,
with only that owner restored alongside the surviving source. Preserve its
ordinary test-only visibility and declaration role. Other rejection owners
are absent; no stub, replacement overload or placeholder value is introduced.
Dependencies on an excluded declaration therefore fail ordinary resolution,
whether from surviving source or from another rejection case.

Every annotation must match exactly one primary error from its own case's
check, by category, code, exact source file and target line. Use the first
character of the primary source range; columns, range ends and messages are
not match keys. An EOF error uses the line containing the EOF position.
Every primary error produced while checking that case must match one of its
annotations exactly once, even if the error originates in another declaration
or dependency. Missing, duplicate, uncoded and unexpected errors fail the case.
Normal successful checking also fails it. Attached notes and related locations
do not count as errors; report warnings without matching them. Error order is
irrelevant. Ordinary recovery need not invent cascading diagnostics.

## Selection and results

The usual test-name selectors select both named executable tests and named
rejection tests, using the existing uniqueness and unknown-name rules. With no
selector, select all named tests. An unknown selector fails admission. Cases owned by other declarations are module
checks and always run. Unselected named rejection tests remain excluded; check
their metadata and boundaries, but do not match diagnostics from their bodies.
`--part` uses the ordinary same-module rules; annotations keep their original
file and line, and surviving helpers are visible across parts.

Invalid metadata, ambiguous isolation or errors in the surviving program fail
admission before any test executes. A rejection-case mismatch instead fails
that case: continue checking other cases and execute selected valid tests if
the surviving program is valid. A failed executable test does not change an
already passing rejection result. Build failures, crashes, timeouts and other
infrastructure failures never count as expected source errors.

Report each selected named case by name and kind, and every declaration-owned
case by source file and declaration-start line, adding its name when available.
The file/line identity distinguishes overloads and malformed declarations.
Distinguish executed-test and rejection-case counts and outcomes; do not report
an excluded test as executed. Exit zero when all required checks and selected
tests pass, or 1 for a reported admission, rejection or test failure.
Infrastructure failures exit nonzero and are identified separately. A run with
no executable tests and at least one checked rejection case is valid. A run
with no cases of either kind retains the existing no-tests behaviour.

Other commands treat expectation annotations as comments and retain ordinary
source checking; only `hgl test` creates rejection cases. See
[mixed examples](../../examples/reject/README.md) and
[conformance cases](../../../compiler/negative_testing/README.md).
