# Ordinary struct constructor order cases

Expected observations under the
[ordinary struct constructor order contract](../language/docs/design/struct-constructor-order.md)
and recursive retention rules VAL-16 and VAL-17. These cases distinguish
expression execution order from the completed value's field association.

## STRUCT-ARG-ORDER — source order and exactly once

Declare Pair with fields left then right, both i64. The example defines
`mark` as an ordinary helper that logs its integer argument and returns it:

```hgl
const fn mark(value: i64) -> i64 {
    inject logger
    info(logger, str(value))
    return value
}
```

The helper is only a way for this test to observe evaluation order; it is not
a built-in or the `record` operator used by `eval`. Evaluating
`Pair(right: mark(2), left: mark(1))` logs 2, 1, exactly once each. The
completed Pair has left equal to 1 and right equal to 2. Reading or retaining
the completed Pair does not repeat either call.

The [source example](../language/examples/struct-constructor-order.hgl)
uses those log messages to check the order. On each admitted evaluation of
construct_pair, its own messages appear as `"2"`, then `"1"`, and it publishes
one complete Pair with the stated fields.

## STRUCT-ARG-RETAIN — retain before continuing

Each supplied argument has two ordered steps: evaluate its expression, then
retain the resulting ordinary value independently for that named field.
Both steps complete before the next expression starts. A supplied aggregate
is recursively retained at that step, not kept as a live alias until all
arguments finish. Later permitted source changes cannot alter that retained
field. The rule grants no otherwise-forbidden mutation, borrow or alias.

A nested constructor completes its own arguments in written order, retains
them and completes its value before the next argument of the outer constructor
starts. Neither inner nor outer field declaration order changes this sequence.

## STRUCT-ARG-FAIL — stop and no completed value

In fail_pair, the right argument first logs 2 and then fails bounds.
The left argument never executes: there is no message `"1"`, no complete Pair
result and no successful constructor publication. The earlier message `"2"`
remains. The evaluation follows the node error contract.

If retaining the first supplied aggregate fails, the same suppression of later
arguments applies, even though its expression succeeded. If a later argument
fails, earlier argument effects remain but their retained fields do not escape
as a partial struct. Final assembly failure likewise supplies no result.

## STRUCT-ARG-CHECK — reject before execution

Unknown or duplicate fields, missing required fields, inadmissible null,
wrong argument types or unresolved constructor type arguments reject the
complete constructor before executing any supplied expression. Thus
`Pair(right: mark(2), unknown: mark(1))` logs nothing and fails checking.
This does not change the ordinary argument or type-checking rules.

## STRUCT-ARG-DEFAULT — omitted constants

For a struct with an effective constant default on left and required right,
construction supplying only `right: mark(2)` calls mark once and places its
result in right. After retaining that supplied result, retain the effective
constant default for left. With several omitted defaults, retain them one at
a time in resolved field declaration order, after every supplied argument
has succeeded. Failure retaining a default stops later default retention and
produces no complete result; earlier argument effects remain. Neither an
omitted constant default nor field assembly invents another call or changes
the supplied expression order.
