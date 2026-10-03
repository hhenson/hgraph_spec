# Library contracts

Status: provisional home, 2026-09-30. The standard library's specification
is its own project; the contracts kept here were derived while triaging
runtime parity issues and are held until that project exists. Nothing here
adds a runtime concept.

Two meanings of "operator" meet in this repository, and each document says
which it means:

| Term | Meaning | Specified in |
|---|---|---|
| **operator** (wiring) | a name that stands for several implementations and is resolved to one of them at each call | [wiring](../wiring/wiring.md), Part 3 |
| **library operator** | one of the operators a standard library provides — `add_`, `union`, `format_`, `default` — and what it publishes as a function of its inputs | [operator contracts](operator_contracts.md) |

A call to a library operator is resolved by the wiring rules like any
other call; what the selected implementation then publishes is its
contract, stated in terms of the runtime rules. The two never overlap: wiring
does not say what an operator computes, and a contract does not say how a
call selects it.

- [operator_contracts.md](operator_contracts.md): admission and nil, set
  operators, formatting and sinks, text, numbers, partitioned dictionaries,
  recording (OP-1 to OP-12).
- [ordinary_replay_record.md](ordinary_replay_record.md): ordinary timed
  scalar data, replay, recording lifecycle and dense eval adaptation.
