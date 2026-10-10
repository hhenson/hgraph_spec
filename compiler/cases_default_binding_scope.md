# Default expression binding scope cases

[DEF-1–4](../language/docs/design/default-binding-scope.md) specify the
scope of existing field and parameter defaults. The
[positive source](../language/examples/default-binding-scope.hgl) exercises
module declarations, formal/field shadowing and generic contextual defaults.
The [standalone negative source](../language/examples/reject/default-binding-scope.hgl)
checks excluded references; its name failures remain uncoded and add no
negative-annotation code.

| Case | Required result |
|---|---|
| Ordinary value-function default calls an enclosing helper with the same name as a formal | Resolve the defining helper for the default; body use of that name reads the formal. |
| Parameter-level `const` configuration has that same spelling | Supplied configuration cannot replace the default's enclosing declaration binding. |
| Earlier/current/later formal is the only declaration of a default's name | Reject name checking; no invocation arguments are available. |
| Earlier instance field is the only declaration of a field-default name | Reject; an instance field is not a constant binding for other defaults. |
| A field name and enclosing helper have the same spelling | The default calls the enclosing helper; supplied instance-field values do not change it. |
| Contextual empty `list<T>` default in two complete generic struct specializations | Check and retain the correct ordinary list type for each specialization. |
| Imported/default specialization has a consumer declaration with the same short name | Preserve the defining binding closure; no caller capture. |
| Default needs an operation or phase absent from its constant context | Existing checking rules reject; the expression grammar adds no permission. |

This matrix claims static source-contract review only, with no HGL target
execution or conformance claim. It defines no new evaluation order, ordinary
capture permission, `sizeof` surface, numeric generic reification, or generic
parameter default syntax.
