# Conformance

Status: proposed methodology; the bounded [dynamic cases](https://github.com/hhenson/hgraph_spec_audit/blob/main/runtime/validation.md) have
recorded Python/C++ runs. Other cases retain their stated evidence limits.

An idle cycle keeps a value but has no delta. Publishing the same value later
can still be a tick. Conformance compares the **trace**, not just the final
value: what a node sees, what runs, and what effects occur, in order.

## A testable rule

Give each rule an ID. Say where it applies, its starting state, what changes,
what stays, when the result can be read, and what happens on failure. Include
empty, invalid and boundary cases. Prose, diagrams and cases must agree.

A precondition limits a case; it does not invent an error outside it. A
rejection has a specified error and post-state. Failure after partial work
also needs cleanup and surviving effects. Cleanup does not imply recovery:
a wiring failure propagates out with no graph description (WIR-4). Suppressing
it and continuing is undefined behavior, with no conformance result to assert.
Omitted observations are untested.

Compare values, validity, modification, deltas, times, bindings, admission,
lifecycle, errors and effects. Include type identity and wiring errors where
specified. Preserve observable order. Addresses, layout and cost belong to a
separate physical contract. Abstract state need not be stored field for field.

An implementation must accept the required programs, reject prohibited ones,
and complete admitted actions under the stated resource assumptions. Where
several outcomes are allowed, list them. An unanswered question is not a
licence for arbitrary behaviour.

## Required and optional behaviour

The rules of the seven chapters are **required**: an implementation that
claims the runtime conforms to every one of them, and every case below is a
check of a required rule. Everything a chapter lists under **Deferred** is
**optional** behaviour (owner, 2026-09-30): observers, externally driven
stepping, one-shot cycle callbacks, the phase hook,
the runaway guard, pausing, checkpointing and recovery, recording and
replay, serial forms, and values owned by a language bridge. An
implementation may provide any of them; when it does, the rules written for
that facility apply, and a conformance claim names which optional facilities
it includes. An optional facility never changes a required observation: a
graph that uses none of them behaves identically on an implementation that
has them all.

[Design options](design_options.md) are neither: they are ways of meeting
the required rules, recorded with their trade-offs, and never a rule.

## Cases

- [Atomic](cases_atomic.md): first tick, idle cycles, equal publications and repeated reads.
- [Collections](cases_collections.md): validity, membership, deltas and per-level time.
- [Windows](cases_windows.md): tick and duration windows, eviction, the minimum.
- [Ordinary struct constructor order](cases_constructor_order.md): argument order, retention and failure.
- [Ordinary list values](cases_ordinary_lists.md): construction, reads, growth and retention.
- [Ordinary scalar replay and recording](cases_ordinary_replay_record.md): timed values, lifecycle and dense horizon.
- [Generator yield operands](cases_generator_operands.md): operand order, target resolution, retention and failure.
- [Ordinary delta types](cases_ordinary_delta_types.md): shape identity, generic matching, storage, construction and publication boundaries.
- [Eval recorder keys](cases_eval_recorder_keys.md): pre-start key selection and source-state isolation.
- [Growing lists](cases_growing_lists.md): growth, truncation, retention and resurrection.
- [Lifecycle](cases_lifecycle.md): activation, construction failure and teardown.
- [References](cases_references.md): sampling, dictionary withdrawal and expiry.
- [Nested graphs](cases_nested.md): keyed routing, state, deadlines and failure.
- [Sources](cases_sources.md): pull sources on either scheduler, push queues and senders.
- [Engine](cases_engine.md): run bounds, ending, the clock and the real-time loop.
- [Injectables](cases_injectables.md): what each injectable shows, and when.
- [Scalar types](cases_scalar.md): identity, capabilities, nil and time arithmetic.
- [Wiring](../wiring/cases_wiring.md): what a generic binds, projections, and operator selection.
- [Representations](representations.md) and [layout](layout_example.md): physical contracts.

Each case names its rules, limits, initial state, actions and observation
points. Times are microsecond offsets from a legal run start unless stated
otherwise; cycles advance strictly. Nil, `never` and zero are distinct.

An **admitted publication** is an event the producer has chosen to publish.
An operator may suppress equal results before that point. The test adapter
must identify the public path that produces the event, drive real endpoints,
and report mismatches without repairing observations.

Use examples, properties over stated domains, differential tests and an
independent implementation. Keep failing seeds and traces. Agreement between
two implementations is evidence, not proof.

Record revisions, rules, cases, adapter, commands, results and untested work.
Keep eligibility, behaviour, physical checks and measured performance separate.
Document checks do not establish runtime conformance.

## Accepting an expectation

State the expected observations and their rule derivation before execution.
Compare each observation with isolated Python and C++ runs:

- Reasoning matches both: accept.
- Reasoning matches either: accept; retain the other result as a variation.
- Python and C++ agree against reasoning: recheck the derivation and adapter.
  Record why an expectation changed; never regenerate it from actual output.
- No pair agrees: ask the user and preserve all three results. A user ruling
  accepts the contract, not a claim that either implementation conforms.
- An explicit user ruling supersedes an earlier acceptance. Retain the old
  expectation, the ruling and both measured results; do not rewrite history.

Python has the stronger testing history and is the preferred guide where
reasoning leaves an interpretation open. Compare fields separately: one
matching field does not validate the whole trace. An unavailable observation,
crash or harness failure is not a matching value. Record the failed phase.

A variation names the case, rules, exact input and observation point, expected
and actual results, implementation identities, repeats and any remaining
uncertainty about the adapter or native core. Keep it until reviewed.

## An implementation card

Name dependencies, deliverables, preserved rules, files, exclusions, checks
and open decisions. For the atomic proof: an isolated owner and action driver;
shared cases; layout and lifetime checks; sanitizers; measured cell costs.
Core integration then needs public C++/Python coverage and the full acceptance
gates. The [layout example](layout_example.md) remains a proposal.
