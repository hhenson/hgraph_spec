# Handler selector cases

These cases specify the clarified handler selection and invalidation behavior.
They are expectations, not compiler execution results. See [handler selectors
and validity handling](../language/docs/user-guide/functions.md#handler-selectors-and-validity-handling)
and [shared source examples](../language/examples/when-defaults.hgl).

## Omitted selectors and explicit validity

These cases compare simple function bodies under different handler selectors.
The first two rows compute `a + b`; the third forwards `a` without adding `b`.
The manual-validity case computes `a + b` when both inputs are valid and returns
`0.0` otherwise. The empty-activation case has a body that returns `0.0`, but
nothing drives it to run.

Use `a: [1.0, _, 3.0]` and `b: [_, 10.0, 20.0]` in fresh runs. Here `_`
means no publication; it does not encode invalidation. Inputs retain valid
values during silent cycles.

| Case / source test | Selection | Required result |
| --- | --- | --- |
| `implicit_and_explicit_defaults_match` | Bare `when` and explicit `modified(a, b) && valid(a, b)` | Both produce `[_, 11.0, 23.0]` |
| `omitted_validity_uses_all_inputs` | `when modified(a)`; validity omitted | `[_, _, 23.0]`; `b` cannot activate the handler |
| `omitted_activation_uses_all_inputs` | `when valid(a)`; activation omitted | `[1.0, 1.0, 3.0]`; `b` can activate it while `a` holds 1 |
| `explicit_empty_validity_leaves_checks_to_body` | `when valid()` with internal guards and a deliberate zero fallback | `[0.0, 11.0, 23.0]`; no implicit requirement blocks the first evaluation |
| `explicit_empty_activation_ignores_input_ticks` | `when modified() && valid()` with no alarm or scheduler | `[_, _, _]`; input publications cannot activate the node, and the compiler should emit a dormant-node warning |

An internal `if valid(b)` does not suppress a missing top-level validity
selector's all-input default. Rejection checks must reject a payload read that
is neither covered by the handler's validity requirements nor guarded by an
internal successful validity check. `valid()` itself proves no input valid.

## Invalidation notifications

Use one handler per function and observe after producers and the consumer have
processed the cycle. An independent observer must inspect output validity and
notification delivery; publication deltas alone do not encode invalidation.
A runtime driver invalidates the source endpoint for these cases; `_` does
not encode that action. Manual output invalidation uses `invalidate(out)`, as
shown below.

| Case | Setup and action | Expected observation |
| --- | --- | --- |
| Managed validity | Single-input function uses `when valid(value)` and previously produced a valid result; its subscribed input becomes invalid | Processing is activated; the result becomes invalid without entering the guarded body. Invalidation notifies and is not a value tick. |
| Manual validity | Single-input function uses `when valid()` and returns `valid(value)`; its subscribed input becomes invalid | The body executes and publishes `false`, even though the input's `modified` observation is false. |
| Unsubscribed invalidation | Two-input function uses `when modified(a) && valid()`; only `b` becomes invalid | No activation occurs solely because `b` became invalid. A later activation from `a` can inspect `valid(b) == false`. |
| Unsubscribed required input | Function uses `when modified(a) && valid(a, b)`; only `b` becomes invalid | No activation occurs solely because `b` became invalid. On a later notification from `a`, the failed requirement for `b` invalidates the output without running the body. |
| Recovery | Managed single-input function returns its input; after invalidation, the input republishes a valid value equal to its old value | The result becomes valid and publishes again; equality with the old invalidated result does not suppress recovery. |
| Silence | A subscribed input remains valid without another notification | Its held value remains available; silence alone neither activates the handler nor invalidates the result. |

### Manual whole-output invalidation

[`manual_forward`](../language/examples/when-defaults.hgl) returns its input
when valid and calls `invalidate(out)` otherwise. Start by publishing `5.0`;
then invalidate the source in a later cycle and observe after the handler runs.

| Action | Expected result |
| --- | --- |
| Source becomes invalid | Output becomes invalid, unmodified, with time `never`; subscribers are notified without a value tick. |
| Source later republishes `5.0` | Output becomes valid and publishes `5.0` again. |
| Call `invalidate(out)` while the output is already invalid | No change or new notification. |
| Write output, then invalidate it in the same evaluation | Final output is invalid; reversing the actions leaves it valid. |
| Attempt `invalidate` on an input or ordinary value, or use it in `start`/`stop` | Reject; this effect requires the injected output during evaluation. |

### Separate handler subscriptions

An outputless function has two handlers: `when modified(a) && valid()` logs
`"a"`, and `when modified(b) && valid()` logs `"b"`. Start with both inputs
valid; in a later cycle, invalidate only `a`. Only `"a"` is logged, even though
both inputs read unmodified. In a fresh run, invalidating only `b` instead logs
only `"b"`. This checks handler dispatch without choosing a shared-output
invalidation policy.

## Empty activation and warnings

An explicit `modified()` selects no input notifications; omitted `modified`
selects every time-series input. A node with no active inputs can still execute
through an alarm or scheduler. Check these diagnostic cases separately from
runtime publication observations:

| Setup | Expected treatment |
| --- | --- |
| No active inputs and no other driver | Compiler warning; source remains valid |
| A scheduler or alarm is injected but never armed from reachable startup work | Warning remains appropriate; capability availability is not an activation |
| The only scheduling call is inside an otherwise unreachable handler | Warning; the handler cannot bootstrap itself |
| No active inputs, but `start` arms an alarm or scheduler and the handler permits that activation | Independent driver exists; an empty input activation set alone does not justify a dormant-node warning |
| Intentional inert terminal such as `null_sink`, without a suppression annotation | Valid source; the dormant-node warning still applies, with no special exemption based on the name or outputless result |
| `null_sink` with `@suppress_warning("node.no_activation")` immediately before its declaration | `node.no_activation` is suppressed for that function; it remains inert and is compiled normally |
| Annotated function also has a different warning | The unrelated warning remains visible |
| Neighboring function or caller has the same dormant-node warning | Its warning remains visible; suppression does not spread to other declarations |
| Annotated function contains a source error | The error remains an error; warning suppression cannot legalize the declaration |

See [selective warning suppression](../language/docs/developer-guide/syntax-and-semantics.md#selective-warning-suppression).
The source example keeps an unannotated `no_input_activation` warning control
beside the annotated `null_sink`. Require the warning code
`node.no_activation` for the former and suppress it for the latter; source
checking must not discard the annotated function as a rejection fixture.
Additional annotation scopes and unknown-identifier handling remain open.
Neither empty selector means the omitted all-input default.

## Deliberate boundary

Combining framework-managed invalidation across multiple ordered handlers that
share an output remains open. These cases do not choose a per-handler output
invalidation policy. Operator-specific departures from the system-library
convention of propagating invalidity must explain their reason and behavior.
