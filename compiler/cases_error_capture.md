# Capture and error-stream cases

Required contract under [CAPTURE-1–5](../language/docs/design/error-capture.md)
and NOD-19/20/29. The [source examples](../language/examples/error-capture.hgl)
and [phase/type rejections](../language/examples/reject/error-capture.hgl)
are normative fixtures, not target execution claims.

For a node accepting nonnegative integers, capture its normal i64 output and
error output. Metadata observers use `modified(step) && valid(step)` so they
run before the error stream is valid as well as after a failure.

| Cycle | Input publication | Value / value modified | Error valid / modified | Held diagnostic |
|---|---|---|---|---|
| 0 | 7 | 7 / true | false / false | absent |
| 1 | -1 | 7 / false | true / true | negative price |
| 2 | _ | 7 / false | true / false | negative price |
| 3 | 8 | 8 / true | true / false | negative price |
| 4 | -1 | 8 / false | true / true | negative price |
| 5 | -1 | 8 / false | true / true | negative price |
| 6 | _ | 8 / false | true / false | negative price |

Every failure has canonical original-node context with phase "evaluate"
and its cycle's logical time. The diagnostic at 5 equals that at 4 as an error
value but still ticks. At 3 success does not clear or republish the last error.
An uncoded failure remains uncoded through capture; no catalogue alias is added.

| Additional case | Required result |
|---|---|
| Node writes out, then ? fails | Both earlier value tick and failure tick stand; subsequent handler statements do not run. |
| Scheduled outer sibling on failed cycle | Its normal evaluation/publication continues in rank order. |
| Whole captured graph fails before a downstream child | Remaining work in that subgraph pass does not run; unrelated outer work continues. |
| Later input or retained future schedule wakes the failed node/graph | Same live instance evaluates; no sticky evaluating state, teardown or automatic retry. |
| Inner capture inside outer capture | Inner failure appears once; outer error stays invalid unless another failure escapes the inner boundary. |
| Consumer ? propagates a retained error on another cycle | New captured event; code/message/original context remain unchanged. |
| Child start/stop fails during a keyed creation/removal evaluation | Failure escapes enclosing capture as a lifecycle failure. |
| Error forwarded as normal data through another graph | Code/message/original context remain intact; no new failure event. |
| Existing keyed body captures each child | Failed key gets its error; successful sibling runs; removal ends only the removed child's error lifetime. |
| New child with same later key | Fresh lifetime/context; no retained failure leaks from removed child. |
| Callable returns result Err normally | Value side publishes Err; capture error side stays unchanged. |
| Captured start or stop failure | Failure escapes; existing startup/cleanup rules apply. Cleanup failure cannot satisfy assert raises. |
| Capture used in value function/handler | Reject capture.phase. |
| Read value field of outputless capture | Reject capture.output_field. |

The source fixture covers node/whole-graph capture, held messages, failure
events, provenance time, nested capture, preserved writes, unrelated work,
keyed composition and uncapturable startup. Future-schedule continuation and
stop/cleanup failure retain the separate runtime lifecycle/scheduler cases;
capture must preserve those contracts, without inventing retry or rollback.

The keyed fixture uses the existing outputless `for key, value in items(book)`
[independent graph-loop contract](../language/docs/design/iteration.md#dynamic-structures-independent-bodies-first).
It requires no new mapping operator, reduction or scalar key-capture rule.
