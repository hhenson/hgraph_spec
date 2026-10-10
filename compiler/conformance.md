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

- [Owned REF output routes](cases_reference_owner_export.md): preserved child
  designations, owner-mediated rank and live boundary composition.
- [Default expression binding scope](cases_default_binding_scope.md): declaration
  closure, excluded argument/instance values and generic contextual defaults.

- [Empty sparse delta application](../runtime/cases_empty_delta_validity.md):
  validity transitions, preserved horizons and [source examples](../language/examples/empty-delta-validity.hgl).

- [Negative testing](negative_testing/README.md): execution-error assertions,
  source-error expectations and bounded controls that must fail.

- [Ordinary tuple construction](../runtime/cases_tuple_construction.md): runtime
  payload construction and isolated constant-context rejections listed in the
  [fixture manifest](tuple_construction/cases.json).

- [Contextual local bindings](cases_contextual_local_bindings.md): fixed scalar
  or connection category, mutability and checking errors.

- [Temporal scalar publications](cases_temporal_scalar_publications.md):
  civil and named-zone values, exact identity and retained publications.

- [Atomic scalar equivalence](cases_atomic_scalar_equivalence.md):
  canonical identity for non-composite types and preservation of composite boundaries.

- [Atomic publications](cases_atomic_publications.md): complete snapshots,
  empty list presence, retention and shape matching.

- [Generic struct shape arguments](cases_struct_shape_arguments.md):
  delta formation, forwarded requirements and ordinary-value restrictions.

- [Retained candidate specialization](cases_retained_specialization.md):
  concrete body/storage substitution without generic value reification.

- [Generator yield operands](cases_generator_operands.md): operand order,
  target resolution, retained payloads and failure under ADR 0015.

The cases retain their source-contract and evidence boundaries. Their placement
does not turn an unmeasured expectation into an observed implementation result.
