# Generator yield cases

These [compiler conformance](conformance.md) cases apply the
[yield rules](../language/docs/design/decisions/0015-pull-sources.md#operand-evaluation-resolution-and-retention).
Let s be the initial evaluation time and d the minimum step. Arithmetic is
representable unless stated; run end follows all future targets. Labels denote
observable helper effects. `after` denotes the statement following a yield.

## YIELD-ORDER — operands and resumption

For `yield time1(2d): value1(1)` followed by `yield time2(d): value2(2)`:

| Time | Effects | Publication |
|---|---|---|
| s | time1, value1 | None; suspend until s+2d. |
| s+2d | after1, time2, value2 | 1; suspend until s+3d. |
| s+3d | after2 | 2. |

Each operand runs once. Each future payload is independently retained;
changing its source before resumption cannot change publication.

## YIELD-PAST and YIELD-STRICT-ORDER — target admission

Each row starts a fresh invocation. Reached yields evaluate time then payload.
An ordering error occurs after those operands, before skipping or publication;
no successor runs. Earlier completed effects and publications remain.

| Resolved targets | Result |
|---|---|
| s-d, s | First skips; second publishes at s. |
| s-d, s-d | First skips; second raises an ordering error. |
| s-d, s-2d | First skips; second raises an ordering error. |
| s-2d, s-d, s | Two skips; third publishes at s. |
| s+2d, s+2d | First publishes on resumption; second raises an ordering error. |
| s+2d, s+d | First publishes on resumption; second raises an ordering error. |
| s+d, s+2d | Both publish at their times. |
| s, s | First publishes; second raises an ordering error. |

The s,s row is YIELD-DUPLICATE. Past pairs also apply before the epoch. Skipping performs no scheduling or
parked copy. A zero duration immediately after future resumption repeats the
preceding target and fails. A fresh invocation has no preceding target.

## YIELD-NEGATIVE and YIELD-FAILURE — error boundaries

| Case | Effects before error | Not performed |
|---|---|---|
| Time expression fails | Earlier time effects | Payload and all later steps. |
| Payload fails, including with a negative duration or past absolute time | time, earlier payload effects | Target validation and all later steps. |
| Duration is negative, initially or after resumption | time, payload | Target addition, scheduling, publication, successor. |
| Explicit time arithmetic overflows inside t | Earlier time effects | Payload and all later steps. |
| Implicit addition of a nonnegative duration overflows | time, payload | Scheduling, publication, successor. |
| Future payload retention fails | time, payload, target validation | Scheduling, suspension, publication, successor. |

Negative-duration admission precedes implicit arithmetic, even if the target
would be representable or addition would underflow. `yield 0s: -1` is valid
as a first yield; the restriction concerns time, not scalar payload sign.
All failures propagate as node errors with no rollback of completed effects
or publications. No failed run is a successful trace. Retention failure is
a required case, not a claim of measured failure injection.
