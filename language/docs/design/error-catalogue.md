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
