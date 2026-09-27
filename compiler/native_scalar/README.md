# Native scalar requirements

[Rules](../../language/docs/design/operators.md#native-scalar-requirements)
separate scalar value capabilities from temporal operators.
[cases.json](cases.json) fixes the observable ticks for representative native
operator implementations; null means no tick. These results also apply to
hgraph_std's `tests/operators.hgl` native-requirement case.

Compiler checks cover exact signature admission, result inference, missing or
ambiguous candidates, execution role, imported identities and capability
propagation. A native requirement neither invokes a temporal operator nor
proves an unrelated scalar expression valid.
