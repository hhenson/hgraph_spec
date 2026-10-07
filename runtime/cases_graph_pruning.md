# Graph pruning cases

Status: reasoned expectations for GRF-26 and WIR-25. These cases specify
observations, not a claim of measured implementation conformance.

## Common setup

All calls wire successfully unless a case explicitly says otherwise. Each
source publishes one valid integer at time 0; each compute forwards it.
Lifecycle observations identify each node's start, evaluation and stop calls.
Sinks record their input publications. The graph runs through time 1 and
stops normally. Stop order is the reverse of start order (GRF-3).

The expected traces below list per-node calls. Ordering between independent
branches is not constrained beyond the graph's rank and lifecycle rules.

## PRUNE-1: unused branch

Wire `live_source -> live_compute -> sink` and separately
`unused_source -> unused_compute`. Neither unused output is exposed.

| Observation | Expected trace |
|---|---|
| live source, live compute, sink | start; evaluate at 0; stop |
| sink publications | `(0, 1)` |
| unused source, unused compute | no instantiation; no lifecycle calls |

An effect inside either unused implementation does not make it a sink.
Run this case both at the root and inside a child graph.

## PRUNE-2: child return and sink

A child wires one source and compute whose output is exposed to its owner,
and a separate source feeding a sink. A third source is unused. The owner
is itself required by a root sink.

| Observation | Expected trace |
|---|---|
| exposed-output source and compute | start; evaluate at 0; stop |
| child sink and its source | start; evaluate at 0; stop |
| owner output and child sink publications | `(0, 1)` at each |
| unused source | no instantiation; no lifecycle calls |

Neither returning an output nor wiring an internal sink discards the other
root. A child that passes an owner input straight through retains that
binding without inventing a child producer.

## PRUNE-3: no roots and shared dependencies

First wire only sources and computes with unexposed outputs and no sinks.
The completed description has zero nodes, and the run has an empty node
lifecycle trace. Then describe a source shared by two sinks. It starts,
evaluates at 0 and stops once, and each sink records `(0, 1)` once. Roots
do not duplicate their shared dependencies.

## PRUNE-4: dependencies independent of scheduling

Repeat PRUNE-1 with a retained node depending on another producer through
each of: a passive input, a feedback input, a member of an assembled
structural input, and an explicitly declared dependency without an input
edge. Every such producer is instantiated, starts and stops. Evaluation
follows the input and scheduling rules; reachability alone does not promise
an evaluation or impose a new rank on feedback inputs.

An unused dependency cycle has no lifecycle trace. A cycle that prevents a
valid rank order among required nodes is rejected; pruning is not a means
of accepting an invalid retained graph.

## PRUNE-5: captures and input positions

A child initially refers to owner inputs A and B. Only B is consumed by a
required child node; A feeds a pruned branch. The completed child retains
only B's binding. At time 0 publish distinct values A=10 and B=20.

| Observation | Expected trace |
|---|---|
| child sink publications | `(0, 20)` |
| A-only child branch | no instantiation; no lifecycle calls |
| B boundary binding | still designates B, not A |

Repeat with B passed straight through as the child's output and no child
nodes. B remains bound; the empty child has no node lifecycle calls.
Implicit closure captures have the same required-only rule as explicitly
listed inputs.

## PRUNE-6: failure is not pruning

Make an invalid call whose output would otherwise be unused. Wiring fails
at the call under WIR-4; no description or run is produced. Separately,
instantiate a completed description containing an unresolved implementation:
GRF-9 rejects it rather than removing the node. Neither case is repaired by
pruning.
