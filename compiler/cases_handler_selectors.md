# Handler selector cases

These cases specify the clarified handler selection and invalidation behavior.
They are expectations, not compiler execution results. See [handler selectors
and validity handling](../language/docs/user-guide/functions.md#handler-selectors-and-validity-handling)
and [shared source examples](../language/examples/when-defaults.hgl).

## Omitted selectors and explicit validity

These cases show how handler selectors control **when a function runs** and
**which inputs it may read**. The functions deliberately use simple bodies so
that the effect of changing a selector is visible. This is not a contract for
one addition operator: three functions add `a + b`, one forwards only `a`, one
adds or returns a fallback, and one never activates from its inputs.

### Functions under test

The declarations below are taken from
[`when-defaults.hgl`](../language/examples/when-defaults.hgl), which also
contains the named tests in the results table. Each function takes two
`f64` time-series inputs and produces one `f64` time-series output.

```hgl
module examples.when_defaults

fn default_sum(a: f64, b: f64) -> f64 {
    when {
        return a + b
    }
}

fn explicit_default_sum(a: f64, b: f64) -> f64 {
    when modified(a, b) && valid(a, b) {
        return a + b
    }
}

fn on_a_when_all_valid(a: f64, b: f64) -> f64 {
    when modified(a) {
        return a + b
    }
}

fn any_input_when_a_valid(a: f64, b: f64) -> f64 {
    when valid(a) {
        return a
    }
}

# The explicit empty validity set admits invalid inputs for internal checking.
fn manual_sum_or_zero(a: f64, b: f64) -> f64 {
    when valid() {
        if valid(a) && valid(b) {
            return a + b
        }
        return 0.0
    }
}

# Expected dormant-node warning: no subscribed inputs or startup driver.
# The warning does not prevent checking or executing this deliberate case.
fn no_input_activation(a: f64, b: f64) -> f64 {
    when modified() && valid() {
        return 0.0
    }
}
```

`default_sum` and `explicit_default_sum` perform the same addition with
implicit and explicit all-input selectors. `on_a_when_all_valid` still adds
both inputs, but only `a` activates it. `any_input_when_a_valid` returns `a`
without reading or adding `b`; `b` can nevertheless activate that handler.
`manual_sum_or_zero` explicitly accepts invalid inputs and chooses `0.0`
unless both values are available. `no_input_activation` would return `0.0`
if driven, but has no input subscriptions or independent driver.

### Input publications and observation

Run each function separately with fresh, initially invalid inputs and output.
Supply these publication sequences:

```hgl
a: [1.0, _, 3.0]
b: [_, 10.0, 20.0]
```

Here `_` means no publication, not zero, an empty value or invalidation. A
previously published value remains valid during silence. At each cycle,
process both input publications before evaluating the function and observe
its output after that evaluation. Both input notifications in one cycle
cause at most one evaluation.

| Cycle | `a` publishes | `b` publishes | Held inputs after producers run |
| --- | --- | --- | --- |
| 0 | `1.0` | `_` | `a` is valid with `1.0`; `b` is invalid |
| 1 | `_` | `10.0` | `a` retains `1.0`; `b` is valid with `10.0` |
| 2 | `3.0` | `20.0` | Both are valid, with `3.0` and `20.0` |

Results below are **output publication sequences**, not snapshots of held
output state. `_` in a result means no output tick in that cycle. Every
reached `return` in these functions publishes its value, including a return
of the same forwarded value as before.

### Expected publications and their derivation

| Source test | Function(s) | Expected publications | Why |
| --- | --- | --- | --- |
| `implicit_and_explicit_defaults_match` | `default_sum`, `explicit_default_sum` | Both produce `[_, 11.0, 23.0]` | At cycle 0, `b` is invalid, so addition is not admitted. At cycle 1, the publication from `b` activates addition using held `a = 1.0`: `1.0 + 10.0 = 11.0`. At cycle 2, `3.0 + 20.0 = 23.0`. |
| `omitted_validity_uses_all_inputs` | `on_a_when_all_valid` | `[_, _, 23.0]` | At cycle 0, `a` activates the function but implicit validity requires `b` too. At cycle 1, only unsubscribed `b` publishes, so there is no evaluation. At cycle 2, `a` activates addition and both inputs are valid. |
| `omitted_activation_uses_all_inputs` | `any_input_when_a_valid` | `[1.0, 1.0, 3.0]` | Only `a` is required valid, and the body returns `a`. At cycle 1, `b` activates the handler through the omitted all-input activation selector, causing another publication of held `a = 1.0`; `b` is not added to it. |
| `explicit_empty_validity_leaves_checks_to_body` | `manual_sum_or_zero` | `[0.0, 11.0, 23.0]` | `valid()` imposes no input validity requirement. At cycle 0, the body runs, finds `b` invalid and returns its explicit zero fallback. At cycles 1 and 2, both checks succeed and the body adds the inputs. |
| `explicit_empty_activation_ignores_input_ticks` | `no_input_activation` | `[_, _, _]` | `modified()` subscribes to no inputs, and no alarm or scheduler drives the function. The body never runs, its `0.0` is never published, and its output remains invalid. The compiler should report `node.no_activation`. |

An internal `if valid(b)` does not suppress a missing top-level validity
selector's all-input default. Rejection checks must reject a payload read that
is neither covered by the handler's validity requirements nor guarded by an
internal successful validity check. `valid()` itself proves no input valid.

## Invalidation notifications

These cases distinguish automatic invalidation from decisions made inside a
handler. Use the following single-handler functions:

```hgl
module examples.handler_invalidation

fn managed_identity(value: f64) -> f64 {
    when valid(value) { return value }
}

fn manual_validity(value: f64) -> bool {
    when valid() { return valid(value) }
}

fn inspect_b_on_a(a: f64, b: f64) -> bool {
    when modified(a) && valid() { return valid(b) }
}

fn sum_on_a(a: f64, b: f64) -> f64 {
    when modified(a) && valid(a, b) { return a + b }
}
```

`managed_identity` forwards its input under a framework-managed validity
requirement. `manual_validity` deliberately publishes whether its input is
valid, including `false` when invalid. `inspect_b_on_a` observes `b`'s validity
only when `a` activates it. `sum_on_a` requires both values, but only `a`
activates addition.

Each row starts a fresh run. Establish the stated initial result before
applying the action. Observe after producers and the consumer have processed
the cycle. An independent observer must inspect output validity and
notification delivery; publication deltas alone do not encode invalidation.
These are runtime-driver cases: the driver invalidates the source endpoint;
`_` in an input sequence cannot perform that action. No new HGL whole-output
invalidation syntax is introduced here.

| Case / function | Initial state and action | Expected observation |
| --- | --- | --- |
| Managed validity / `managed_identity` | Publish `value = 5.0`, producing `5.0`; then invalidate `value` | Processing is activated; the output becomes invalid without entering the guarded body. Invalidation notifies and is not a value tick. |
| Manual validity / `manual_validity` | Publish `value = 5.0`, producing `true`; then invalidate `value` | The body executes and publishes `false`, even though the input's `modified` observation is false. |
| Unsubscribed invalidation / `inspect_b_on_a` | Publish `a = 1.0`, `b = 7.0`, producing `true`; then invalidate only `b`; later publish `a = 2.0` | Invalidating `b` alone causes no activation, so the output holds `true`. The later publication from `a` runs the body, which observes invalid `b` and publishes `false`. |
| Unsubscribed required input / `sum_on_a` | Publish `a = 1.0`, `b = 7.0`, producing `8.0`; then invalidate only `b`; later publish `a = 2.0` | Invalidating `b` alone causes no activation, so the output holds `8.0`. On the later notification from `a`, the failed requirement for `b` invalidates the output without running addition. |
| Recovery / `managed_identity` | Publish `5.0`, invalidate the input and process that invalidation, then publish `5.0` again | The output first becomes invalid, then becomes valid and publishes `5.0` again. Equality with its old invalidated value does not suppress recovery. |
| Silence / `managed_identity` | Publish `5.0`, producing `5.0`; advance one cycle with no input action | The output holds valid `5.0` with no new publication or invalidation notification. |

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
