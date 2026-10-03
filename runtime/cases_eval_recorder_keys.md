# Eval recorder-key cases

These expected observations exercise
[eval recorder-key ownership](../language/docs/design/eval-recorder-keys.md).
Keys below are ordinary strings; no particular internal spelling is required.
All required keys and exact types are resolved before root graph start.

## EVAL-KEY-SOURCE — source requirements take part in selection

The target has a source key k of the same ordinary recording type as eval's
output recorder. It writes its own list in start, on each input publication,
or in stop. Eval must choose a recorder key unequal to k. For pass-through
input `[1, 2]`, its output remains `[1, 2]`; the target's entry retains the
value dictated by its ordinary writes. Neither source nor recorder writes
are redirected to the other's entry.

Repeat with a source key whose type differs from the recording type. Eval
still selects another string before binding its recording. It does not
manufacture an avoidable type conflict by choosing the already-used name.

A read-only requirement for k also excludes k, including a get whose branch
never executes. The recorder must not initialize a source-owned absent
entry simply because the target has not written it. This case imposes no
requirement to execute that get successfully while the entry is absent.

## EVAL-KEY-SEED — seeded values are preserved

A run owner supplies k with an ordinary typed value before eval finalizes
its seed configuration and selects internal recorder keys.
Whether or not the target otherwise uses k, eval's recorder key differs.
With no source writes to k, its supplied value survives recorder start,
publication and stop. An empty or all-silent eval still initializes its own
empty recording and leaves the seed untouched.

This does not waive ordinary seed/type compatibility when the target itself
requires k with a different type. Such a source conflict still fails under
the existing checking/construction rules; eval does not rename user keys.

## EVAL-KEY-SEED-CLOSED — finalization precedes selection

Finalize the owner-supplied seeds, then select recorder key r. Before root
start, attempt to add a further seed at r with the exact recording type.
This fails as an eval configuration error before any start hook runs; the
late value is not installed, and recorder-key selection is not silently
repeated. Repeat with a different late key: it is the same closed-phase error,
not a check that only recognizes collisions with r.

Replacing a previously supplied seed after finalization also fails, even
with the same key and type. Changing the original value owner after successful
configuration cannot change the independently retained seed. These cases do
not add rollback of earlier successful configuration effects.

A separately configured eval may use different seeds. Outside eval, the
general run-owner seeding window in ADR 0016 remains before graph start;
ordinary start/evaluation/stop writes to prepared source entries remain legal.

## EVAL-KEY-NESTED — the closed inventory includes later use

A known nested graph requires source key k, but is instantiated only after
an input tick, or its access lies in a branch that never runs. Its requirement
still participates in the run's existing pre-start entry preparation. Eval
selects another key before root start. Later ordinary access to k cannot
reach the recorder's entry by a same-string collision.

An unknown key computed from runtime child data remains outside the closed
profile; no late runtime renaming or new dynamic-key facility is inferred.

## EVAL-KEY-MULTIPLE — internal keys and independent runs

Every eval-owned recording key differs from all other internal keys required
or selected for its run. After choosing one recorder key, selection of the
next excludes it as well as all source and seed keys. Matching ordinary
payload types do not permit sharing a recording entry.

Separate runs own separate stores and may reuse identical internal strings.
Results and source state do not accumulate across those runs implicitly.
No test depends on a particular generated prefix or spelling.

## EVAL-KEY-ORDINARY — no change to caller-selected keys

Two ordinary source nodes deliberately using the same key and exact type
still access the same entry in execution order. An independent caller of
record does not acquire a private namespace. Incompatible source types at
one key still fail before start. This extension adds no writer restriction,
rollback or runtime key/role check to any of those cases.
