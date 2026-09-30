# Open points

Status: register opened 2026-09-30. Every chapter keeps its own "Points to
settle"; this page lists the ones still open in one place so that they can be
ruled on together, and records the rulings of 2026-09-30. A settled point
stays in its chapter with its ruling.

## Settled on 2026-09-30 (owner)

| Point | Ruling | Where |
|---|---|---|
| A reference carried backward through a feedback | A feedback carries values only; a reference is always of lower rank than its consumer | TS-28; Graph point 9; TS point 4 |
| How long a removed thing is kept | Finalised at the end of its cycle; reclaimable from the start of the next; later at the implementation's option | Overview rule 15; GRF-25; Graph point 10 |
| Wiring and operators in the runtime chapters | Wiring is specified apart from the runtime; operators are wiring (resolution) and library (contracts); the runtime starts at the builder boundary | Overview; `wiring/`; `library/` |
| Required versus optional behaviour | Chapter rules are required; deferred facilities are optional and named in a conformance claim | Conformance |
| Implementation choices | Recorded as design options with rationale and trade-offs, never as rules | Design options |
| Structural inputs | A third input activity: membership changes wake the node, value ticks do not | NOD-28; Node point 5 |
| Nested error capture | A nested node captures a child's failure in its own eval, keeps the child, and may capture per key | NOD-29; Node point 6 |
| Compatibility at a binding | The table in Time-series, "What may bind to what" | TS-29; TS point 5 |
| The alarm | The one-shot scheduler is a distinct concept: no state, no recovery, no selector | NOD-25 to NOD-27; Injectables |

## Still open

Each needs an owner ruling or a case. The options are as the chapter states
them.

| # | Point | Chapter | The question |
|---|---|---|---|
| 1 | How an implementation is named | Graph 1; Overview 3 | Namespace, supply of implementations before instantiation, generic implementations: one identity per resolved type, or identity plus type arguments. Part of the builder-boundary discussion |
| 2 | What implements an HGL body | Graph 2; Overview 3 | Compiled code registered under an identity, or the body carried in the description |
| 3 | How types appear in a description | Graph 3; Overview 3 | By value, as a table, or by canonical name; and the closed set of an abstract family's members |
| 4 | Is input peering stored or derived | Graph 4 | Derivable from edges except for local inputs and run-time members |
| 5 | Output mode | Graph 6 | Part of the node description, or implied by the child graph's output binding |
| 6 | Scheduling in the past | Graph 7; Node 3 | The graph errors, hgraph's node scheduler ignores; GRF-12 errors. Should the scheduler be as strict? NOD-25 already makes the alarm strict |
| 7 | A cancelled request still wakes the node | Graph 8 | Harmless but observable; accept, or require the schedule entry be withdrawn |
| 8 | Inputs in start and stop | Node 1 | HGL forbids; is that the runtime's rule or the language's |
| 9 | The output on a cycle that fails | Node 2 | What was written stands (NOD-19), or no tick |
| 10 | Where the push queue's type is recorded | Node 4 | Part of the implementation identity, or a description item checked at instantiation |
| 11 | Failure handling order | Engine 1 | Specify stop-then-report only and defer the alternative, or keep both as configuration |
| 12 | Pausing a cycle, for mesh | Engine 2 | An optional facility with rules, or a mesh design without it |
| 13 | When is an owned bundle valid | TS 1 | Output stays valid after its last field invalidates; a non-peered input does not |
| 14 | The delta of a whole value | TS 7 | `delta_value(x)` has a specified scalar result; structural contextual result types and output application remain open |
| 15 | Writing the output in start | Injectables 1 | Nothing needs it; the rule to change is INJ-8 |
| 16 | `lag` on HGL's clock | Injectables 2 | A language extension; lands in hgraph first |
| 17 | Integer overflow, division, NaN | Scalar 1 | hgraph's arithmetic table is the reference until HGL settles it |
| 18 | Strings | Scalar 2 | Ordering, indexing, length, normalisation undefined in HGL |
| 19 | An unknown enum integer | Scalar 3 | Number fallback (hgraph) or rejection (HGL); a runtime receiving one from outside needs one answer |
| 20 | Clearing an optional field through a delta | Scalar 4 | No spelling for "now unset" in a bundle delta |
| 21 | Field order with several abstract parents | Scalar 5 | Undecided in both hgraph and HGL |
| 22 | Map equality | Scalar 6 | The registry requires key equality and hash only; the value type presumably needs equality too |
| 23 | The key set as a structural observation | Node 5 | Whether a bound key set and a structural input on the dictionary observe the same thing; a case decides |

## Decisions taken elsewhere that these chapters follow

- 2026-09-19: `all_valid` is one level, never recursive (TS-9).
- 2026-09-21: a saved reference expires at the next cycle boundary (TS-23).
- 2026-09-24: the first admitted set result publishes even when empty; a
  map's text order is unspecified (library, OP-5, OP-9).
- 2026-09-26: bundle identity, requested references, run-time element
  selection, a wiring failure fails the graph, a candidate never widens its
  operator (WIR-4, WIR-12, WIR-15, WIR-21 to WIR-24).
- 2026-09-29: the library constant is spelled `const`; the alarm does not
  answer `scheduled()` (HGL ADR 0015, MIG-009).
