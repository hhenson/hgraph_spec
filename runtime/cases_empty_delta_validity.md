# Empty sparse delta cases

Status: accepted expected behavior, 2026-10-10; no runtime measurement is
claimed here. Rules: [EMPTY-1–4](../language/docs/design/empty-delta-validity.md).
Observe state after the producer using an independent step input, so silent
applications remain observable. Check valid, modified, last-modified time,
child validity, values and membership separately from recorded deltas.

## EMPTY-INITIAL — EMPTY-1, EMPTY-4

Begin with an invalid endpoint. For a list use a zero-length growing list
in the first trace. E denotes its explicit empty delta; D is a canonical nonempty update. The output receives these applications
at successive times; a pass-through applies its input delta to its own output.

| Input application | Output valid | Output modified | Recorded delta |
|---|---|---|---|
| E | yes | yes | E |
| E | yes | no | none |
| D | yes | yes | D |
| E | yes | no | none |
| no application | yes | no | none |

The first application initializes no children or memberships; a fixed shape's
children remain invalid. Silent rows retain the preceding modification time
and all state. An eval with this five-slot input returns `[E, _, D, _, _]`:
suppressed applications do not shorten the horizon. Test sets, maps, zero-length
growing lists, tuples and named structs. For admitted zero-child shapes omit D;
first E ticks, later E is silent, and the valid empty endpoint is all-valid.
Applying E twice within one evaluation preserves the first tick and its time.

For a positive fixed list, or a growing list resized to positive length with
no valid positions, the same five inputs give `[_, _, D, _, _]`. E keeps
every position invalid and the parent time *never*. D writes a position,
making the parent valid. See the [list validity trace](cases_growing_lists.md#list-empty-publication--ts-9-ts-32).

## EMPTY-NESTED — EMPTY-2

Start with a fixed parent whose growing-list children are unpublished.
Target one child with an empty delta. The child and its ancestors become
valid and modified. The other children stay invalid. Record the parent patch with its explicit empty
child entry. Repeat it on the next cycle: no tick. An empty patch to the valid
parent does not initialize its remaining invalid children. Target another
unpublished growing-list child with its own empty delta: it becomes valid and ticks
the parent. Also test a new map key or contiguous growing-list position whose
structural child is initialized by E; repeats preserve membership and are silent.
For a fixed list child of positive length, E leaves all positions invalid.
An existing field receiving only this child edit stays quiet until a position publishes.
Invalid removals, duplicate additions, gaps and out-of-range entries remain errors.

## EMPTY-REVALIDATE — EMPTY-1

A producer establishes validity, then undergoes existing runtime invalidation.
An independent observer sees the invalid state and time *never*. Applying E
subsequently revalidates a zero-length growing list with an empty tick at
that cycle's time. For a fixed list of positive length E leaves it invalid;
a position publication revalidates it. Other structural shapes that admit
empty initialization follow the first trace. Repeating E is silent. The
observer reads the producer after each operation.

## EMPTY-CANCELLATION — EMPTY-3

A set producer adds then removes the same member in one cycle. Its tick and
empty net delta remain observable under TS-10. Apply that delta to separate
outputs: an invalid output ticks empty and becomes valid; an already valid
output retains its state and time without ticking. Do not infer a consumer
tick from the producer's modified state alone.

## EMPTY-COMPLETE-CONTROLS — EMPTY-4

Two empty atomic list snapshots cause two publications. Two empty ordinary
list arrivals to a rolling window likewise cause two arrivals. Repeated scalar
zero, false and empty text still tick. These are complete payloads, not sparse
empty patches. The [HGL examples](../language/examples/empty-delta-validity.hgl)
cover initial/repeated application, nested application, invalid nonempty
instructions after an empty application, and these controls;
state-only and invalidation cases remain independently observed as above.
