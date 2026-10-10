# Required-constant Any capability diagnostic

The [source example](../language/examples/reject/any-constant-capability.hgl)
combines rejected named defaults with surviving positive tests. It applies
existing ANY-3/5 required-constant checking under the
[`value.constant_capability` catalogue row](../language/docs/design/error-catalogue.md#source-errors).
No boxing, field/index access, constant-variable syntax or key admission is added.

| Case | Required result |
|---|---|
| Known missing contained order in struct default, including ordinary list index or struct field extraction | Source category `type`, code `value.constant_capability`, at the required operation expression. |
| Known missing contained key equality/hash in struct default after field extraction | Same source category/code at the key expression. |
| Known missing contained capability in a value-function parameter default | Same required-constant checking result. |
| Ordinary Any numeric order and empty Any key in required defaults | Check successfully; surviving positive tests execute. |
| Corresponding comparison in an executed test/runtime body | Retain execution code `value.capability`; optional folding cannot turn it into source rejection. |
| Earlier type, phase, constructor, index or field formation error | Preserve the applicable checking failure; no cascade or recategorization is required. |

The source and execution codes remain disjoint phase identities, not aliases.
Unknown capabilities do not trigger this new stable diagnostic merely because
execution could fail. Constant evaluator implementation and HGL conformance
are not claimed by these source cases.
