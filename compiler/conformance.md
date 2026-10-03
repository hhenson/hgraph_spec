# HGL compiler conformance

These cases check HGL source-language behavior and its lowering. They are
separate from [runtime conformance](../runtime/conformance.md): a runtime-only
implementation need not accept HGL source or implement its generator syntax.

A compiler claiming the linked HGL source contract must preserve its observable
behavior when lowering to runtime facilities. For example, a generator's
operand order and suspension belong to the compiler's implementation of
ADR 0015; the resulting nodes and scheduling still obey their runtime rules.
This does not add generator machinery to the runtime contract.

## Cases

- [Retained candidate specialization](cases_retained_specialization.md):
  concrete body/storage substitution without generic value reification.

- [Generator yield operands](cases_generator_operands.md): operand order,
  target resolution, retained payloads and failure under ADR 0015.

The cases retain their source-contract and evidence boundaries. Their placement
does not turn an unmeasured expectation into an observed implementation result.
