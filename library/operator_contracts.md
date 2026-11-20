Library operator contracts
==========================

Status: draft, 2026-09-24; moved from the runtime chapters to the library on
2026-09-30 (owner), provisionally: the standard library's specification is
its own project ([README](README.md)). This chapter covers the
operator families below. The rest of the library remains open (OPEN-10, OPEN-11 in
[Beyond the runtime core](../runtime/boundaries.md)).

"Operator" in this document means a **library operator**: one of the
operators the standard library provides, such as `union` or `format_`. A
**contract** says what such an operator publishes as a function of its
inputs, stated in terms of the runtime chapters: once wired, a library
operator is a node or a graph of nodes and obeys the node and time-series
rules like any other. How a call to it is resolved to an implementation is
the wiring concept of an operator ([wiring](../wiring/wiring.md), Part 3)
and is not repeated here. The library is not the runtime: nothing here
adds a runtime concept, and a contract that needs one names the missing
component in the runtime's points to settle.


Admission and nil
-----------------

An operator's inputs are admitted by NOD-2: by default every input must be
valid before the operator is evaluated. An invalid input is nil (TS-2). It is
not an empty collection, a zero or a default. A set that has never ticked
is not the empty set; it is no set at all.

System-library operators propagate invalidity to dependent outputs. For an
operator with one or more time-series inputs, all inputs being invalid is
sufficient to invalidate its output; its contract may require invalidation
sooner. This convention also applies when `when valid()` leaves validity
handling to the body. An exclusion must document its behavior and justification
in the operator's contract. This is an authoring rule, applied through the
existing activation subscriptions, not unconditional framework enforcement.
Sources with no time-series inputs are outside this rule.

A contract may name inputs that the operator reads while they are invalid.
It must then say what an invalid input contributes. `union` reads the valid
operands and ignores invalid ones, but its output is invalid when none is
valid. `format_` with `__strict__` false admits invalid arguments, subject to
the same all-inputs-invalid rule unless its contract documents an exclusion.
`is_empty` and `mean`, including dictionary `mean`, propagate input invalidity;
invalid input produces neither `True` nor NaN. An empty valid collection is a
separate case.

A non-peered fixed collection is valid while any child is valid. An aggregate
over a fixed list is therefore admitted once one element is valid, and reads
the valid elements only (the accepted valid-subset reduction).

An operator never changes what it reads. Reading a view that an output
derives from itself, such as a key set's emptiness, must not make that
output, or its owner, tick (NOD-5, NOD-21, TS-21).

An operator that **derives** a value may elide an unchanged recomputation.
An operator that **forwards** a value, routing one input's ticks to its
output, publishes every tick it forwards, equal or not: the tick is the news
(ruling 2026-07-17).

Scalar map lookup and the scalar-container reductions `min_`, `max_`, `sum_`
and `mean` may take a default. A missing map key
or an empty container selects the default when it has a value. If the map
contains the key, or the container is nonempty, clearing the default alone
leaves the result in place. For a one-element integer tuple containing the
lookup result, `min_`, `max_` and `sum_` publish the same trace. The mean
publishes those values as floats.

| Cycle | Map publication | Default | Lookup publication for key `x` |
|---|---|---|---|
| 1 | `{x: 5}` | `8` | `5` |
| 2 | — | clear | — |
| 3 | `{x: 5}` | — | `5` |
| 4 | `{x: 6}` | — | `6` |


Set operators
-------------

`union`, `intersection`, `difference` and `symmetric_difference`, and their
spellings `|`, `&`, `-` and `^`, act on TSS values and on TSD key sets.

| Operator | Result members | Admission | A TSD output child forwards |
|---|---|---|---|
| union | in any operand | the valid operands | the operand whose child ticked most recently; leftmost on a same-cycle tie |
| intersection | in every operand | every operand valid | lhs |
| difference | in lhs, not in rhs | both valid | lhs |
| symmetric difference | in an odd number of operands | every operand valid | the one operand holding it; for three or more, the pairwise left fold |

The first admitted evaluation publishes the result even when it is empty:
an admitted result is a value, and valid does not mean non-empty (TS-1;
owner ruling 2026-09-24). After that a change that nets to nothing does not
tick.

A member entering the result, or a TSD child whose forwarding operand changes
because another operand lost the key, takes the new operand's current value.
That is a derived change: an equal value is not published again, unless
the new operand's child also ticked in this cycle, which is forwarded.

`union`, `intersection` and `symmetric_difference` fold pairwise over any
number of operands; `difference` takes exactly two. An empty intermediate
dictionary result follows the first-admitted-result rule (OP-5).


Formatting and sinks
--------------------

`format_` with `__strict__` true, its default, publishes nothing until every
argument is valid. There is no placeholder for a missing argument. `print_`,
`log_` and the formatting `assert_` format their arguments exactly as
`format_` does with its defaults, so none of them emits a line before every
argument is valid.


Merge
-----

`merge` combines ordered streams of the same time-series type. Its
**original sources** are the ordered input streams supplied to that call.
Invalid sources do not supply a value. When several valid sources publish
in the same cycle, the leftmost one supplies the output, including an explicit
publication equal to the previous output (OP-4).

If no valid source publishes in that cycle, a fallback selects the valid
source with the most recent modification time. This is the time of the
**original source** for that call; selecting a held value does not refresh
that source's modification time. Equal times are resolved leftmost. A
fallback publishes only when its value differs from the current output, or the output has no value
yet. With no valid source, the output is invalidated rather than retaining a
previous value. Invalidation is a notification, not a value tick or a removal.

A separately nested call, such as `merge(merge(A, B), C)`, has the inner
merge's output as one original source of the outer call. A tick published by
the inner call is a current-cycle source publication for the outer call,
even when the inner call derived it by fallback. Calls are not flattened;
nested calls need not have the same trace as `merge(A, B, C)`.

Ordinary dictionary merge maintains the union of its inputs' live keys,
independently of child validity. Adding a key makes the output dictionary
valid; a newly introduced child remains invalid until a valid source supplies
a selected value. Membership alone supplies no child payload. Apply value
selection independently to each key's original children, so different keys
from different inputs can publish together. With no valid source for a live
key, that output child is invalidated rather than retaining its previous value.
Child invalidation does not remove the live key. If all input dictionaries are
invalid, the whole output is invalidated.
Removing a key permits fallback to another input that still holds it. The
key remains in the output until the last input removes it (TS-19).
This contract does not specify the separate `disjoint=True` optimization.

[Merge cases](cases_merge.md) give the input publications and required traces.


Associative list reduction
--------------------------

Associative reduction over a growing list uses its currently valid positions.
With two or more positions it combines their values. With one position and a
supplied zero it combines that value with the zero. With no live positions
left after positional removals, it forwards the supplied zero or withdraws
its result when zero was omitted. Positional removals retire leaves even when
clearing the last valid child makes the list itself unmodified (TS-32).
Repeating a removal of an already vacant position is silent. A subsequent
write restores that position as a new operand. Restoring a checkpoint between
these events yields the same outputs as uninterrupted execution.

For addition, begin with positions 0 = 3 and 1 = 4 and zero 0. Remove both
positions while zero publishes 5, remove position 0 again, publish zero 5
again, then write position 0 = 8. The result publishes 7, 5, nothing, 5,
and 13. If zero stays silent during the first removal, its retained value is 0
and the second result is 0.


Text
----

String conversion follows the [HGL conversion contract](../language/docs/user-guide/types-and-expressions.md#string-conversion).
A top-level string is its own text. Members of a set or map appear in no
specified order.

`repr` is not part of the standard; implementations may provide it as an
extension. Container string quoting and escaping rules remain open.


Numbers
-------

`ln` of zero is negative infinity and of a negative number is NaN: the IEEE
results.


Partitioned dictionaries
------------------------

`unpartition` flattens a dictionary of dictionaries, `TSD[K1, TSD[K, V]]`,
into one dictionary `TSD[K, REF[V]]`: each inner key `k` of each partition
becomes an output key holding a reference to that partition's child. The
partition that last published `k` **owns** it; the partitions' key spaces
are expected to be disjoint, and which partition owns a key published by two
in one cycle is unspecified.

- An inner key removed from its partition is removed from the output.
- **Removing a partition removes every key it owns from the output, in the
  same cycle** (owner ruling 2026-09-26). A consumer of the output sees those
  keys removed (TS-19); an input bound through one of the references unbinds
  (TS-15), and a dictionary input reports the key removed.
- Removing a partition costs the keys that partition owns, never the size of
  the output.

Dynamic schedules
-----------------

For a schedule with a time-series delay and optional time-series start,
a new start publication re-anchors the schedule and restarts its tick
budget. Clearing the start between alarms leaves the pending alarm and
remaining budget intact. If the clear coincides with a due alarm, the
schedule consumes that alarm once and continues at its delay.

With an initial five-microsecond delay, start at offset zero and four
permitted ticks, clearing start at offset two yields ticks at offsets
5, 10, 15 and 20 microseconds. Clearing at the first alarm yields the
same trace. Publishing a fresh start of two microseconds at offset two
instead yields ticks at 7, 12, 17 and 22 microseconds.

Window resets
-------------

A count-based or duration-based `to_window` clears its samples when
`modified(reset)` is true. It clears before adding a simultaneous price.
A direct set or dictionary clear wakes the node with an unmodified reset
input (TS-7, TS-18). The window retains its samples on that wake.

Start with prices 10 and 20 in a window with room for all three prices and
no expirations during the trace.

| Reset action | Price in the same cycle | Window event | Next price | Next window event |
|---|---|---|---|---|
| Invalidate the reset collection | — | — | 30 | [10, 20, 30] |
| Invalidate the reset collection | 30 | [10, 20, 30] | — | — |
| Publish the reset collection | — | [] | 30 | [30] |
| Publish the reset collection | 30 | [30] | — | — |

A valid empty publication also resets the window when it modifies the reset input.


Recording
---------

A recording exists from the moment its recording node starts. A recorded
time-series that never ticks leaves an empty recording, which is distinct
from no recording at all; replaying it publishes nothing.

For record/replay comparison, a wake that finds both operands invalid adds
zero comparisons, zero mismatches and zero recorded comparison rows.
A valid value on just one side is a mismatch, including when the other
side is explicitly cleared. When both sides hold values, comparison on
a publication still uses their current values, including equal repeated
publications.

| Left operand | Right operand | Comparison result |
|---|---|---|
| Publish 7 | Publish 7 | Match |
| Clear | Clear | No comparison |
| Clear again | Clear again | No comparison |
| Publish 7 | Publish 7 | Match |
| Clear | Hold 7 | Mismatch |

The first four steps produce two comparisons and zero mismatches for two
independently driven operands or one stream connected to both inputs.

The language's [eval form](../language/docs/design/eval-operator-composition.md)
wires replay source operators and a record sink operator, owns their typed
buffers and run-specific configuration, and captures owned copies of output
deltas. Eval callers do not configure these operators separately; this does
not restrict their independent library APIs. Materializing a dense eval
result from an empty recording preserves its input horizon with no-tick
cells; it does not publish those cells as ticks.

The proposed [eval configuration foundation](../language/docs/design/decisions/0016-eval-scalar-buffer-capabilities.md)
passes replay its sequence as ordinary const data. Record has the ordinary
contract `record<T>(ts: T, const key: str)` and requests reusable run-wide
`global_state`. The [ordinary scalar data contract](ordinary_replay_record.md)
supplies `TimedValue<T>` with `time: datetime` and `value: delta<T>`,
`replay<T>(const values: list<TimedValue<T>>) -> T`,
and scalar record bodies using ordinary list construction and push. The
[ordinary delta type extension](../language/docs/design/ordinary-delta-types.md)
generalizes the same data path to admitted structural publication deltas.
No specialized storage capability supplies it.



Rules
-----

- **OP-1** Unless its contract names an input it may read while invalid, an
  operator propagates invalidity when any input is invalid. An operator with
  one or more time-series inputs propagates invalidity when all are invalid,
  even if none is required valid by its handler. Exclusions must document
  their behavior and justification. Propagation follows activation subscriptions
  and invalidates a previous output; silence alone is insufficient. A nil input
  is not an empty collection, a zero or a default.
- **OP-2** `is_empty` and an aggregate over a collection — `sum_`, `mean`,
  `min_`, `max_`, `all_`, `any_`, `len_` of a set — follow OP-1, including
  dictionary `mean`. A default value answers an empty valid collection, not an
  invalid one. An aggregate over a fixed list reads its valid elements once any
  element is valid.
- **OP-3** An operator never changes an input or its producer. A derived view
  of an output — a key set's emptiness, a membership test — is read without
  ticking that output.
- **OP-4** An operator that forwards a value publishes every tick it forwards,
  equal or not. An operator that derives a value may elide an unchanged
  recomputation.
- **OP-5** A set operator's members are those in the table above. A TSD output
  child forwards the operand the table names (OP-4); a derived change of
  forwarding operand publishes only a different value. The first admitted
  evaluation publishes the result, empty or not.
- **OP-6** `union` reads the valid operands and ignores an invalid one;
  when all operands are invalid, its output is invalid.
  `intersection`, `difference` and `symmetric_difference` follow OP-1.
- **OP-7** `union`, `intersection` and `symmetric_difference` fold pairwise
  over any number of operands; `difference` takes exactly two.
- **OP-8** `format_` with `__strict__` true, and `print_`, `log_` and the
  formatting `assert_`, produce nothing until every argument is valid.
- **OP-9** The members of a map or a set appear in no specified order in text.
  Container string quoting and escaping remain open. `repr` is an
  implementation-specific extension, outside the standard.
- **OP-10** `ln` of a non-positive float is `-inf` for zero and NaN below zero.
- **OP-11** A recording exists from its recording node's start; one that
  recorded no tick is empty, not absent.
- **OP-12** `unpartition` publishes each partition's inner keys as references
  to their children. A key leaves the output when it leaves its partition, or
  when the partition that owns it is removed; the latter costs the removed
  partition's own keys.
- **OP-13** A `merge` call's original sources are its ordered input streams;
  a nested merge output is a source, not a flattened list of its inputs.
  `merge` ignores invalid sources. The leftmost valid source
  modified in the current cycle supplies the value and its publication is
  forwarded, even when equal. Otherwise select the valid original source
  with the latest modification time, leftmost on ties, and publish only a
  changed value or a first value. With no valid source, invalidate the output.
  Ordinary dictionary merge applies this rule per live key, invalidating a
  child without removing its key when no source child is valid. It maintains
  the union of live input keys independently of child validity and invalidates
  the whole output when all input dictionaries are invalid. Adding a key makes
  the parent valid, with an invalid child until a value is selected; the last
  input's removal removes the output key.
- **OP-14** For scalar map `getitem_` and scalar-container `min_`, `max_`,
  `sum_` and `mean`,
  invalidating an unused default alone produces no publication. A simultaneous
  map, lookup key or container publication is processed normally, including
  an equal repeat.

A TSD delta holding a nested dictionary includes the inner dictionary's delta:
its valid modified children only. An invalid inner child contributes nothing,
never a default value. This is the TSD row of the value and delta table in
[Time-series types](../runtime/time_series.md); no new rule is needed.


Points to settle
----------------

1. **The first empty result.** Settled 2026-09-24 by the owner: yes, the
   first admitted evaluation publishes an empty result (OP-5). The reasons:
   validity means having a value, and an admitted result has one; the four
   set operators should not differ by how they are built; and without it
   `symmetric_difference(a, b, c)` never publishes when `a` and `b` cancel,
   although its value is well defined.
2. **The order of a map's text.** Settled 2026-09-24 by the owner: a map is
   an unordered map and provides no ordering guarantee, so the order of its
   text is unspecified (Scalar types, Text; OP-9). A set has no order either
   (VAL-8) and is treated the same way.
3. **Invalid inputs.** Settled 2026-10-18 by the owner: `is_empty` and
   `mean` propagate invalidity. `union` and `merge` propagate when all inputs
   are invalid. This is the general system-library convention, including for
   handlers with no required-valid inputs; exclusions require a documented
   justification (OP-1).


Implementation source references are recorded in the
[audit notes](https://github.com/hhenson/hgraph_spec_audit/tree/main/docs/implementation-notes).
