# Generator yield operand cases

These are [HGL compiler conformance](conformance.md) cases for generator
lowering, not additional required rules for a runtime-only implementation.

These expected observations clarify
[yield operand evaluation](../language/docs/design/decisions/0015-pull-sources.md#operand-evaluation-resolution-and-retention).
Let s be the current body evaluation time and d the minimum engine step.
All example target arithmetic is representable unless the case says otherwise.
The run end is later than all supplied future targets.

An operand may log its label using an already admitted value helper. Labels
in these cases name observations, not new HGL primitives. A label after a
yield means execution reached the next statement in the generator body.

## YIELD-ORDER — exactly once, before suspension

The first yield logs `time1` while returning 2d, then logs `value1` while
returning 1. The next statement logs `after1`; a second yield logs `time2`
while returning d and logs `value2` while returning 2. Its successor logs
`after2`.

| Evaluation time | Effects, in order | Publication |
|---|---|---|
| s | time1, value1 | none; target s+2d is scheduled |
| s+2d | after1, time2, value2 | 1 |
| s+3d | after2 | 2 |

Neither operand is evaluated again at its yield's resumption. The second
relative duration is resolved from s+2d. Each parked payload is independently
retained before suspension; changing its former source cannot change the
eventual publication.

## YIELD-PAST — operands execute, publication skips

A yield whose time operand returns the absolute instant s-d logs `time1`
then `value1` and immediately continues to `after1`, without publishing or
scheduling that entry. The same applies to a duration -d whose resolved
target is s-d. A following `yield 0s: 2` publishes 2 at s; the skipped entry
has not consumed a publication at that time. No parked retained payload is
required for the skipped entry.

This also applies at the initial run evaluation to an absolute pre-epoch
instant, or a representable negative relative duration. Generator skipping
occurs before any scheduling request, so it does not relax the scheduler's
separate refusal of past requests.

## YIELD-FAILURE — operand and arithmetic boundaries

| Failure | Effects before failure | Not performed |
|---|---|---|
| Time expression logs time then fails | time | Payload evaluation, scheduling, publication, successor statement. |
| Payload expression logs payload then fails, including for a past target | time, payload | Target resolution, scheduling, publication, successor statement. |
| Explicit datetime-plus-duration inside the time operand is unrepresentable | Earlier time-expression effects | Payload evaluation, scheduling, publication, successor statement. |
| Implicit relative-target addition is unrepresentable | Both successful operand effects | Scheduling, publication, successor statement. |
| Retaining a future payload fails | Both successful operand effects and target resolution | Scheduling, suspension, publication, successor statement. |

The failure propagates as a node error. Completed earlier effects remain.
Neither checked arithmetic failure wraps into a past target to be skipped.
Retention failure is a required ownership/error case, not a claim that
failure injection has been measured.

## YIELD-DUPLICATE — error after evaluating the second pair

At s, a first due yield evaluates time1 then value1 and publishes 1. Its
successor runs. A second due yield evaluates time2 then value2 and resolves
to s again, then fails as a duplicate publication. Its successor does not
run; the second payload does not replace the first. This does not roll back
the first publication or earlier operand effects. The enclosing run reports
an error, not a successful trace with a silent second entry.

These cases specify source behavior independently of any implementation.
