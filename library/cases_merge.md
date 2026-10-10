# Merge cases

Status: accepted contract, 2026-10-10. These are expected traces for OP-13,
not implementation algorithms.

Each table starts a fresh ordinary dictionary merge. Inputs A, B, C and D
are ordered left to right and hold `TSD[str, TS[int]]`. Times are consecutive
engine cycles. Unless labelled as state observations, input and output cells are **deltas**,
not complete dictionary values: `k:1` publishes the value 1 at key k, `k:R` removes k, and `none`
means no publication. Observe after producers and merge have evaluated.

## MERGE-ORIGINAL-RECENCY: OP-13

| t | A | B | C | Required output |
|---|---|---|---|---|
| 0 | k:1 | k:2 | k:3 | k:1 |
| 1 | none | none | k:33 | k:33 |
| 2 | none | k:22 | none | k:22 |
| 3 | none | k:R | none | k:33 |

At 0 the same-cycle tie selects A. At 3 B no longer holds k. C last
published at 1, later than A at 0, so C supplies 33. Selecting a fallback
does not change any input's modification time.

## MERGE-TIES-AND-EXPLICIT-EQUAL: OP-4, OP-13

| t | A | B | C | Required output |
|---|---|---|---|---|
| 0 | k:1 | k:2 | k:3 | k:1 |
| 1 | k:1 | none | none | k:1 |
| 2 | k:R | none | none | k:2 |
| 3 | none | none | none | none |

At 1 an explicit equal publication is forwarded. At 2 B and C have the
same original modification time, 0; B wins by argument order. An idle cycle
does not republish a held value.

## MERGE-EQUAL-FALLBACK: OP-4, OP-13

| t | A | B | C | Required output |
|---|---|---|---|---|
| 0 | k:1 | k:1 | k:1 | k:1 |
| 1 | k:R | none | none | none |
| 2 | none | k:1 | none | k:1 |

At 1 B replaces A but the value is still 1, so the derived fallback is
silent. At 2 B explicitly publishes 1; that publication is not elided.

## MERGE-UNSELECTED-REMOVAL: OP-13

| t | A | B | C | D | Required output |
|---|---|---|---|---|---|
| 0 | k:1 | k:2 | k:3 | k:4 | k:1 |
| 1 | none | k:20 | none | k:40 | k:20 |
| 2 | none | none | k:30 | none | k:30 |
| 3 | none | k:R | none | none | none |
| 4 | none | none | k:R | none | k:40 |

At 3 removing unselected B leaves the output value unchanged. At 4 D last
published at 1 and A at 0, so D supplies the fallback. This call reads all
four original inputs; a fallback does not refresh their modification times.

## MERGE-NESTED-CALLS: OP-4, OP-13

Compare a single three-input call with two explicitly nested calls. Let
M be `merge(A, B)`, F be `merge(A, B, C)` and N be `merge(M, C)`.

| t | A | B | C | M | F | N |
|---|---|---|---|---|---|---|
| 0 | k:1 | k:2 | none | k:1 | k:1 | k:1 |
| 1 | none | none | k:3 | none | k:3 | k:3 |
| 2 | k:R | none | none | k:2 | none | k:2 |

At 2 M derives a fallback from B and publishes 2. That tick is a current-cycle
publication from N's input M, so N forwards it. F still reads A, B and C
individually: C is newer than B, and its held value 3 equals F's current
value, so F publishes nothing. A separately nested call introduces an
observable publication boundary.

## MERGE-KEYS-AND-LAST-REMOVAL: TS-19, OP-13

| t | A | B | C | Required output |
|---|---|---|---|---|
| 0 | none | none | none | none |
| 1 | k:1 | j:2 | none | k:1, j:2 |
| 2 | k:R | none | none | k:R |
| 3 | none | j:R | none | j:R |

At 0 no source has supplied a value. At 1 each key is selected separately,
so both publications reach the output. At 2 and 3 no input still holds the
removed key; the output removes it. Removing the last output key is a
removal delta, not silence and not an empty-dictionary publication.

## MERGE-INVALID-CHILD: TS-19, OP-13

Run both variants. At cycle 0, A publishes only the valid sibling `"j":9`,
or remains invalid without publishing an empty dictionary. B then introduces
`"k"` with a never-ticked child. This is a membership operation, not a nil
payload publication. Observe the merged output's state after evaluation;
delta-only recording is insufficient.

| t | B operation | `"k"` live | `"k"` added | `"k"` removed | Root valid | Live `"k"` child valid | `"k"` child payload delta |
|---|---|---|---|---|---|---|---|
| 1 | Add `"k"` without a child value | yes | yes | no | yes | no | nil |
| 2 | None | yes | no | no | yes | no | nil |
| 3 | Publish 7 to `"k"` | yes | no | no | yes | yes | 7 |
| 4 | Remove `"k"` | no | no | yes | yes | — | — |
| 5 | None | no | no | no | yes | — | — |

With a sibling, live keys are `{ "j", "k" }` in cycles 1–3, then
`{ "j" }`; `"j"` stays valid with value 9. Without a sibling they are
`{ "k" }`, then empty. At cycle 3 the child's value is 7 and no membership
addition recurs. At cycle 4 the output delta removes `"k"`; `—` means no
live child, without changing the removed-child observation rules (TS-11).
Forwarding only valid child payloads fails at cycle 1: `"k"` must already
exist, even though it contributes no child payload delta.
