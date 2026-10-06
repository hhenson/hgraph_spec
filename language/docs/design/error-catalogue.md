# Error catalogue

Codes identify a specified failure independently of diagnostic text. The tables
enumerate all codes admitted by negative-test expectations. They are an initial
catalogue, not a claim that every possible failure has a stable code. An
unlisted failure still fails normally and cannot satisfy a coded expectation.
Code spelling and meaning are language contracts; messages may supply context.

## Execution errors

Only these codes are accepted by `assert raises`:

| Code | Required failure |
| --- | --- |
| `yield.negative_duration` | After both yield operands succeed, its duration operand is negative. |
| `yield.non_increasing_time` | After operand evaluation and target resolution succeed, the yield target is equal to or earlier than the preceding target in this generator invocation. Skipped past targets count. |

The [yield operand order](decisions/0015-pull-sources.md#operand-evaluation-resolution-and-retention)
determines which failure occurs. Operand failures and arithmetic overflow are
not renamed to either code. A negative duration fails before target resolution
or comparison. A first yield has no preceding target; an otherwise valid first
past absolute target is skipped, not an ordering failure.

Codes remain attached when an execution error propagates out of a graph to an
evaluation caller. Wrapping an error must preserve its original code. This
does not change which graphs capture errors as output data.

## Diagnostic categories

These are all diagnostic category names. Categories group problems; codes
identify particular failures. Only listed source codes may be expected in
a source-rejection case. Recognizing a category does not make every failure in
that category available to coded expectations.

| Category | Meaning |
| --- | --- |
| `parse` | Source does not conform to lexical or grammatical syntax. |
| `name` | A name is undefined, duplicated or otherwise invalid in its scope. |
| `type` | A value, argument or type application violates its required type or value domain. |
| `shape` | A temporal or ordinary structural shape is not admitted at a boundary. |
| `constraint` | A declared generic requirement cannot be satisfied or resolved consistently. |
| `function-kind` | A function body or signature violates the rules of its function kind. |
| `phase` | A construct or capability is unavailable in its containing phase or context. |
| `injectable` | An injected capability is unknown, duplicated or incompatible with its declaration. |
| `operator` | An operator call or implementation has no compatible resolution. |
| `module` | A module, import, export or visibility boundary is invalid. |
| `build` | Artifact construction or its infrastructure failed after source checking; not a source-rejection category. |

## Source errors

These are all codes accepted by `# expect-error`. The category in the same row
is required. Other source failures may remain uncoded. When a listed condition
occurs, it must carry the listed category and code. Preserve ordinary checking
order and diagnostic recovery; the catalogue does not require cascading errors
after an earlier failure prevents checking a construct.

| Category | Code | Required failure and primary location |
| --- | --- | --- |
| `parse` | `syntax.expected_token` | A mandatory grammatical token is missing. Locate the unexpected token encountered in its place, or EOF when input ends. |
| `type` | `rolling.size_kind` | A resolved rolling maximum or minimum is neither `i64` nor `duration`, or the two sizes have different kinds. Locate the offending size argument; for mixed kinds, the minimum. |
| `type` | `rolling.size_bounds` | Sizes have valid matching kinds, but a tick size is nonpositive, a duration maximum is nonpositive, a duration minimum is negative, or minimum exceeds maximum. Locate the offending size argument; for minimum exceeding maximum, the minimum. |
| `type` | `yield.time_type` | A yield time operand is neither `duration` nor `datetime`. Locate that operand. |
| `type` | `test.raises_code` | A syntactically valid `raises` argument is not a single string literal naming a catalogued execution code. Locate the argument. Check its source form before constant folding. |
| `phase` | `test.statement_phase` | A named test or its nested raises block contains `state`, `cache`, `inject`, `start`, `when`, `stop`, `yield` or `return`. Locate the forbidden statement's keyword. |

Malformed syntax inside a `raises` argument is a parse error, not
`test.raises_code`. In particular the grammar admits an expression there but
the static rule requires a literal. The two catalogues are disjoint: an
execution code cannot be used as a source-error code or conversely.
