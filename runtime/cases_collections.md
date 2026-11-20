# Collection cases

Status: QUOTE-HISTORY, MEMBERSHIP-INDEPENDENT and SAME-CYCLE-RESTORE are
established cases. SET-WHOLE-AND-SPARSE, COLLECTION-INVALIDATE-AND-REWRITE
and SPARSE-CHILD-PUBLICATION describe the expected observations for explicit
producer writes. The remaining cases are proposed unless separately identified.

## VALIDITY-IMMEDIATE — TS-9

The root is valid; each inner list has two elements.

| Root | Immediate children | Descendants | Root all_valid |
|---|---|---|---|
| TSL of two lists | first valid; second invalid | first only partly populated | false |
| TSL of two lists | both valid | both only partly populated | true |
| TSD of lists | live X valid | X only partly populated | true |
| TSD of lists | live X invalid | no valid value in X | false |
| TSD after removing X | no live children | removed X does not count | true |

Check the root's validity and immediate children's **valid**, never their
`all_valid`. A never-published empty collection fails. The invalid-child row
also rules out the older shortcut `TSD.all_valid = valid`.

## SET-DELTA — TS-5 and TS-10

Start with an invalid `TSS<str>`. At 10 add A and B: value and added are
`{A,B}`, removed is empty, valid and modified are true. At 20 add C and remove
A: value `{B,C}`, added `{C}`, removed `{A}`. Idle at 30: retain the value,
unmodified, no delta. Repeated reads agree. In a separate cycle, adding then
removing a new D leaves it in neither delta set; the set still ticks.

## SET-WHOLE-AND-SPARSE

Start with an invalid owned set. At each step a producer explicitly writes
the indicated value or delta. A separate clock drives an observer after the
producer so it can distinguish a tick with an empty delta from silence.

| Step | Producer operation | Held members | Modified | Membership delta |
|---|---|---|---|---|
| 1 | Publish the whole set containing A | A | true | added A |
| 2 | Publish the same whole set | A | true | empty |
| 3 | Apply an empty membership delta | A | false | no tick |
| 4 | Publish the whole empty set | empty | true | removed A |
| 5 | Publish the whole empty set again | empty | true | empty |

Each publication advances the last modified time. On step 3 that time stays
at step 2. In Python, a `frozenset` result supplies the whole value and
`set_delta()` supplies the sparse control. These are producer writes;
derived set operators decide when to write under their own contracts.

## COLLECTION-INVALIDATE-AND-REWRITE

On successive steps, establish each owned output, invalidate the whole
output, then apply the sparse write below. Observe after each step.

| Output | First value | Sparse write after invalidation | Recovered value |
|---|---|---|---|
| TSD | keys A = 1 and B = 2 | write C = 3 | only C = 3 |
| TSS | members A and B | add C | only C |
| Growing TSL | positions 0 = 10 and 1 = 20 | write 0 = 30 | length 1, position 0 = 30 |

On the middle step the parent is invalid and unmodified, its last modified
time is *never*, and its value and delta are nil (TS-1, TS-2, TS-7).
The rewrite makes it valid and modified. Compare clearing only the child
at key B of a dictionary: B remains a member with an invalid child.

Also exercise invalidation and a rewrite **in the same producer step**.
A dictionary starts with A = 1 and B = 2. A mapped child adds 10 to each
value. The producer invalidates the dictionary and writes C = 3 before
returning. A downstream observer sees only C = 13 in the mapped result,
with removals for A and B. On the next step, invalidate the dictionary
without rewriting it. The source is invalid and the mapped result reports
C removed. Writing A = 4 in a later step yields only A = 14.

## SPARSE-CHILD-PUBLICATION

Use an owned dictionary of quote bundles with fields `bid` and `ask`.
An empty patch to a new key A makes A a member, initializes its bundle and
ticks the dictionary. Apply the same patch again: its child and the dictionary
are quiet. Write bid = 7 at A, then repeat the empty patch. The bid remains
7 and both last modified times stay at the bid write. This is an instance
of [EMPTY-NESTED](cases_empty_delta_validity.md#empty-nested--empty-2).

A sparse bundle patch may identify a field by name or by its declared
position. Each supplied field applies its own delta in both forms. Exercise
both spellings on fresh outputs; sample the child and parent separately.

| Bundle field and initial value | Named patch | Positional patch (field 1) | Observation |
|---|---|---|---|
| `book`, dictionary with A and B | `{"book": {"A": REMOVE_IF_EXISTS}}` | `(None, {"A": REMOVE_IF_EXISTS})` | Only B remains. Child and parent tick, and the child reports A removed. |
| `members`, valid set containing A | `{"members": set_delta()}` | `(None, set_delta())` | A remains. Neither the child nor the parent ticks. |

In these Python spellings field 0 is an ordinary scalar and is omitted by the
patch. Its value and last modified time are preserved.

## QUOTE-HISTORY — TS-1 through TS-5, TS-9, TS-11 and TS-19

`TSD<str, TSB{bid: TS<i64>, ask: TS<i64>}>`, initially invalid and empty.
Only X; at most one field publication or removal per cycle. Publication creates
X if absent. Empty insertion, field invalidation, multiple keys, same-cycle
remove/reinsert, references, failures and borrows are excluded.

Begin retains membership, values and last times; modified is false and changes
are empty. Read after each action below; 40 is idle.

| Time/action | X present | bid, ask | Field delta | Membership added / removed | Root / row / bid / ask last times |
|---|---|---|---|---|---|
| initial | false | —, — | — | empty / empty | never / — / — / — |
| 10: publish bid 7 | true | 7, — | bid 7 | X / empty | 10 / 10 / 10 / never |
| 20: publish ask 9 | true | 7, 9 | ask 9 | empty / empty | 20 / 20 / 10 / 20 |
| 30: publish bid 7 | true | 7, 9 | bid 7 | empty / empty | 30 / 30 / 30 / 20 |
| 40: idle | true | 7, 9 | — | empty / empty | 30 / 30 / 30 / 20 |
| 50: remove X | false | —, — | — | empty / X | 50 / — / — / — |
| 50: observe again | false | —, — | — | empty / X | 50 / — / — / — |
| 60: publish bid 7 | true | 7, — | bid 7 | X / empty | 60 / 60 / 60 / never |

After removal, `—` means no current child. Both reads at 50 see the removed
child `{bid: 7, ask: 9}`, with row/bid/ask times `30 / 30 / 20` (TS-11).
That removed view expires at the cycle boundary; reinsertion at 60 is fresh.

Initially all flags are false. Thereafter the root is valid and all_valid,
including when empty; modified is true except at 40. A present row is valid;
its all_valid is false at 10 and 60, true at 20–40, and absent at 50.

Both [representations](representations.md) must retain the equal tick at 30,
the ask's own time, repeated removal reads, and a fresh ask at 60.

## MEMBERSHIP-INDEPENDENT — TS-19

Insert a key with an invalid child: the key is added. Publishing or invalidating
that child later neither adds nor removes the key. Removing the key does.
Replay needs both membership changes and published-value deltas (TS-5).
This case extends beyond QUOTE-HISTORY.

## SAME-CYCLE-RESTORE — TS-11

Publish X=7 at 0. At 1 remove X, inspect the removed child twice, then restore
X without publishing a value. Both removed reads give 7 with child time 0;
restoration returns that value and time. The final added and removed sets are
empty. The dictionary has still ticked. Remove at 2 and insert X=9 at 3:
this is a new child.
