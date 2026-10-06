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
| `eval.input_delta_profile` | While executing `eval`, a supplied input trace violates its publication profile. Validate after evaluating its ordinary arguments and before starting any graph; identify the parameter and zero-based position, retaining the `eval: input delta outside publication profile` message prefix. |
| `yield.negative_duration` | After both yield operands succeed, its duration operand is negative. |
| `yield.non_increasing_time` | After operand evaluation and target resolution succeed, the yield target is equal to or earlier than the preceding target in this generator invocation. Skipped past targets count. |

The [yield operand order](decisions/0015-pull-sources.md#operand-evaluation-resolution-and-retention)
determines which failure occurs. Operand failures and arithmetic overflow are
not renamed to either code. A negative duration fails before target resolution
or comparison. A first yield has no preceding target; an otherwise valid first
past absolute target is skipped, not an ordering failure.

`eval.input_delta_profile` is an eval admission failure, catchable by `assert raises`
before graph execution. It does not turn source-checking or build failures into
execution errors. Constructor formation and exact-type checking happen first.
The code covers only violations already excluded by the publication profile;
it chooses no empty-event, invalidation or redundant runtime mutation semantics.

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
| `shape` | `delta.unsupported_shape` | A resolved concrete T in `delta<T>` has no admitted delta type, including after generic substitution. Locate T in the delta application. |
| `type` | `delta.type_mismatch` | A supplied expression fails the existing assignment/contextual typing rules for an explicitly required `delta<T>`, or a structural delta is supplied to an incompatible ordinary destination. Locate the supplied expression. Structural originating shapes must agree exactly; reduced scalar, atomic and rolling types retain their ordinary value rules. |
| `name` | `delta.argument_name` | A structural delta constructor uses an unknown argument or field name. Locate that name. |
| `name` | `delta.duplicate_argument` | A structural delta constructor repeats an argument or field name. Locate the later name in written order. |
| `type` | `delta.entry_constant` | A constructor member, key or index expression is not an admitted constant or cold recipe. Locate that expression. |
| `type` | `delta.entry_type` | A constructor member, key or index has the wrong exact ordinary type. Locate that expression. |
| `type` | `delta.duplicate_entry` | Two entries within one constructor argument have equal members, keys or indices, with equality known during checking. Locate the later member/key/index expression in written order. |
| `type` | `delta.index_bounds` | A correctly typed constant list index is negative, or a fixed-list/tuple index is outside its shape. Locate the index expression. Growing-list removal bounds depending on preceding length are eval admission checks. |
| `type` | `delta.overlap` | Set added/removed members or map upsert/remove keys overlap, with equality known during checking. Locate the member/key expression in the later written argument. Growing-list removed/modified overlap is an eval admission check. |
| `phase` | `test.statement_phase` | A named test or its nested raises block contains `state`, `cache`, `inject`, `start`, `when`, `stop`, `yield` or `return`. Locate the forbidden statement's keyword. |

Malformed syntax inside a `raises` argument is a parse error, not
`test.raises_code`. In particular the grammar admits an expression there but
the static rule requires a literal. The two catalogues are disjoint: an
execution code cannot be used as a source-error code or conversely.

For these delta codes, formation precedes use, and entry typing and constant
admission precede equality/bounds checks that require them. No cascade is
required when an earlier failure prevents a later check. Provider-dependent
key identity remains deferred to cold materialization when its run context is
needed; `delta.duplicate_entry` and `delta.overlap` do not require guessing it
at checking time. This catalogue adds no general constructor-execution code.
See the [delta failure fixtures](../../examples/reject/delta-errors.hgl) and
[eval admission fixtures](../../examples/eval-profile-errors.hgl).
