# Library contracts

Status: provisional home, 2026-09-30. The standard library's specification
is its own project; the contracts kept here are held until that project
exists. Nothing here
adds a runtime concept.

Two meanings of "operator" meet in this repository, and each document says
which it means:

| Term | Meaning | Specified in |
|---|---|---|
| **operator** (wiring) | a name that stands for several implementations and is resolved to one of them at each call | [wiring](../wiring/wiring.md), Part 3 |
| **operator behavior** | what an operator publishes as a function of its inputs | [operator contracts](operator_contracts.md) |

A call to an operator is resolved by the wiring rules like any
other call; what the selected implementation then publishes is its
contract, stated in terms of the runtime rules. The two never overlap: wiring
does not say what an operator computes, and a contract does not say how a
call selects it.

`core` owns core operator declarations and the baseline implementations required
from every engine. Standard-library operators belong in `std`, which may also
extend operators declared in `core`; see the [core operator policy](../language/docs/design/operators.md#core-operators-and-the-standard-library).
This directory records behavior contracts; a contract's placement here does
not decide whether its operator belongs to the core or the standard library.

- [operator_contracts.md](operator_contracts.md): admission and nil, set
  operators, formatting and sinks, text, numbers, partitioned dictionaries,
  recording and merge (OP-1 to OP-13).
- [cases_merge.md](cases_merge.md): original-source fallback recency,
  same-cycle and fallback ties, equal publications, invalid-child membership,
  dictionary removals and separately nested calls.
- [ordinary_replay_record.md](ordinary_replay_record.md): ordinary timed
  scalar data, replay, recording lifecycle and dense eval adaptation.
- [Ordinary delta types](../language/docs/design/ordinary-delta-types.md)
  extend the timed data relationship to the admitted structural profile.
- [Atomic publications](../language/docs/design/atomic-delta-publications.md)
  add complete finite snapshots to the same replay and record operators.
