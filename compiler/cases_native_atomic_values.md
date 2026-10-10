# Native atomic value cases

These check [NVAL-1–5](../language/docs/design/native-atomic-values.md).
Each target provides checked implementations of the shared HGL interface.
Use an owning text-backed Token with equality/hash/order, a different nominal
TokenOther with the same logical layout, and TextOnly lacking those optional
capabilities. These names and capabilities are fixture contracts, not built-ins.

| Case | Required result |
|---|---|
| Export/import `native type` and exact helper signatures | Preserve canonical nominal identity and provider/capability metadata. |
| Two HGL bindings map to one existing native scalar identity | Aliases preserve one canonical type; no duplicate native type is created. |
| Different native canonical identities have identical layouts | Distinct types, including when boxed. |
| Missing provider, mismatched identity/capabilities/signature | Fail before execution, without inferring a mapping from target source. |
| HGL fields, inheritance, generic application or type-name construction | Checking failure. |
| Helper called in executed test setup/cold materialization | Execute only descriptor-permitted calls without runtime-only capabilities; never run native code during source checking. |
| Typed native atomic get and local var replacement | Get owns an independent copy; replacing the local does not write its global entry. |
| Ordinary helper result, mutable binding replacement and retained old copies | Independent owning values; same canonical type. |
| Required native capability absent | Checking failure for the known native type; boxed executed use fails with `value.capability`. |
| Token equality, order, boxed Token versus TokenOther | Supported content operations; different nominal identities never collapse. |
| Equal/distinct values, leading/interior/trailing silence and empty input | Pass-through preserves publications and dense eval horizon. |
| `atomic<Token>`, `delta<Token>`, `TimedValue<Token>` | One native scalar identity and complete delta. |
| Native leaf in sparse fixed/growing list, tuple, struct and map | Preserve outer shape/omissions; retain exact native payload identity. |
| Native complete compound value, box and rolling arrival | Independent complete copies and existing arrival rules. |
| Native keys and set members | Exact equality/hash, duplicate/overlap checking, removal and reinsert lifecycle. |
| Timed replay and record; source replacement; fresh runs and teardown | Preserve values/times without aliasing or premature provider destruction. |
| TextOnly passed through then projected by its checked text helper | Copy/transport needs no invented equality/hash/order. |
| Opaque RAII resource state as a temporal payload | Checking failure; NVAL does not broaden the state bridge. |
