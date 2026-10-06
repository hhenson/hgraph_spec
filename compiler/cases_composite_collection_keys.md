# Finite composite collection key cases

| Case | Required result |
|---|---|
| Equal independent tuple or concrete struct keys | Address the same member under full value equality. |
| Equal hash with unequal contents | Keep distinct keys. |
| Same-layout unrelated struct supplied as K | Reject exact nominal mismatch. |
| Partial delta used as a composite key | Reject; a key is a complete ordinary value. |
| Duplicate equal key or added/removed overlap | Reject before any target starts. |
| Provider-dependent component | Retain exact cold result; validate deferred duplicates before start. |
| Optional component unset versus present zero | Preserve distinct key identity. |
| Source binding mutates after ordinary key capture | Preserve stored identity and prior recordings. |
| Removal and later reinsertion | Preserve normal membership lifecycle and full key identity. |
| Atomic set/map with composite keys | Preserve complete replacement and present empty snapshots. |
| Collection-containing, recursive, abstract, opaque or reference key | Remain outside this admission. |
