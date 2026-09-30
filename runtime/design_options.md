# Design options

Status: recorded 2026-09-30 (owner direction). Nothing here is a rule.

A rule says what a test can observe. A design option is a way of meeting the
rules that an implementation may take or leave, recorded so that the
reasoning behind hgraph's choices is not lost and so that a new
implementation can weigh the same trade-offs. Where a rule fixes a minimum
(the earliest a removal may be reclaimed, GRF-25), an option describes what
an implementation may do beyond it. An option never changes an observation;
if it would, it is a rule and belongs in a chapter.

Each option names the rules it serves, what it buys, and what it costs.

## Delayed reclamation

Rules served: GRF-25, TS-11, TS-23, TS-32.

The minimum is that a removed thing stays readable to the end of its cycle
and is reclaimed no earlier than the start of the next. hgraph goes further
and reclaims lazily: a removed dictionary child, a truncated list tail or a
window's evicted value is left in place until the collection is next
touched, and the physical erase happens then. Deltas are read through
*modified*, so nothing needs sweeping at the cycle boundary (TS chapter, "A
tick").

- Buys: no end-of-cycle pass over every collection; removal is a flag and a
  retained slot, not a free; a key re-inserted soon after reuses its slot.
- Costs: memory is held past its logical lifetime; a collection that is
  removed from and never touched again keeps its tail; expiry of a saved
  reference must be observable at the cycle boundary regardless (TS-23), so
  a lazily reclaimed slot still needs a generation or an equivalent guard
  against a reference attaching to reused storage.

## Two address spaces for a switch

Rules served: GRF-23, GRF-25, GRF-2.

A switch that replaces its branch stops the old graph and starts a new one.
The minimum lets the old graph be disposed of any time after the cycle.
hgraph keeps the old branch until the next switch, and an implementation may
go further and hold two fully built address spaces, A and B, alternating
between them: the outgoing branch stays intact while the incoming one is
built, and a switch back reuses what is already there.

- Buys: no allocation on the switch path once both spaces exist; a branch
  that flips back and forth (an on/off condition) pays nothing per flip;
  readers of the outgoing branch's outputs in the switching cycle are
  trivially safe.
- Costs: twice the storage for the lifetime of the switch; state in the
  reused space must still be rebuilt, since a stopped graph is never started
  again (GRF-20); with more than two branches the scheme degrades to a
  cache with an eviction policy.

## Cached observations on assembled inputs

Rules served: TS-27, TS-3, TS-26.

A non-peered collection input derives its validity, modification and time
from its children. It may recompute them on every read, or maintain them
from child events. hgraph maintains them: a child notification updates the
parent's change time, and a read costs nothing beyond a field.

- Buys: reads that do not scan children; an assembled parent with thousands
  of children reads as cheaply as a scalar.
- Costs: every child event must reach the parent, including invalidation
  and rebind; the cache must reset when the whole structure becomes invalid
  (TS-26) and when a rebind changes a child's target (TS-25). A bug here is
  invisible to a test that only reads once.

## Sharing an assembly node instead

Rules served: TS-27.

The alternative to an assembled input is one node that builds the
collection and owns an output, to which every consumer binds. hgraph folds
the assembly into the consumer when there is one consumer, and may share a
node when there are several.

- Buys, for the fold: no extra node, no extra output, no copy of the
  children's values.
- Buys, for the shared node: one set of observations kept once; consumers
  bind to a peered output and need no local state.
- The choice is a cost choice; the observations must be the same either
  way (TS-27).

## No state slot for the alarm

Rules served: NOD-25, INJ-4.

A node that only ever asks to be woken keeps nothing between evaluations.
hgraph gives such a node no scheduler storage at all: the alarm is a view
over the graph's schedule, constructed on demand, and the node's memory
holds no request.

- Buys: the smallest possible source node; nothing to record or restore.
- Costs: nothing to cancel, query or tag; a source that later needs any of
  those changes to the scheduler, which is a change to what it declares.

## Placeholders at a nested boundary

Rules served: GRF-10, GRF-24.

A child graph's boundary inputs are bound directly to the outputs the
owner's inputs are bound to, with nothing in between. hgraph compiles the
child against placeholders for those inputs and binds them at instantiation,
rather than inserting a forwarding node.

- Buys: a child reads the real producer; a tick reaches it with no extra hop
  and no copy.
- Costs: a rebind of the owner's input must reach every child that captured
  it, live, without recreating the child's state (GRF-10); the placeholder
  must preserve peering and reference routes recursively.

## Generations on reclaimable slots

Rules served: TS-23, GRF-25.

Where storage is reused, a saved reference to something removed must not
find the newcomer. hgraph guards reuse with an invalidation subscription;
the Rust store stamps each slot with a generation that a reference carries
and checks.

- Buys: a reference check is a comparison; reuse is free.
- Costs: a generation is one more word per slot and per reference, and it
  can wrap; wrap handling needs its own test.

## How to add one

Name the rules the option serves, so that a reader can tell it changes no
observation. Say what it buys and what it costs. If a case would observe the
difference, it is not an option.
