# Empty sparse delta application

Status: accepted, 2026-10-10.

**EMPTY-1.** Applying an explicit empty sparse `delta<T>` to an invalid
endpoint makes that endpoint valid and modified at the current evaluation
time, with an empty delta. It initializes no child, member or key. Applying
it to an already valid endpoint causes no tick and changes no value,
validity, membership or modification time. It does not clear a tick already
caused by an earlier operation in the same cycle.

This applies to sets, maps, fixed and growing lists, tuples and named structs
in the admitted [structural profile](contextual-collection-deltas.md), including
zero-child shapes where their type and delta constructors already exist.
This does not admit `tuple<>` or `()`, which the current grammar excludes.
Construction or retention alone publishes nothing. No new constructor syntax
or invalidation operation is introduced.

**EMPTY-2.** Apply the rule recursively at each explicitly targeted child.
A nonempty parent patch containing an empty child delta ticks when that child
becomes valid; the parent's delta retains that present empty child entry.
Omitted children remain unchanged. Repeating the patch when none of its
entries causes a publication is silent. Canonical membership, index and
removal checks still apply: invalid nonempty instructions are errors, not
empty updates. A new map or growing-list child may receive an empty structural
delta that makes it valid, but creating an invalid child remains excluded.

**EMPTY-3.** An empty net delta may accompany an original producer tick
following same-cycle cancellation. That producer tick stands. Applying its
empty payload to another valid endpoint is silent; applying it to an invalid
endpoint follows EMPTY-1. Payload application does not copy event identity.

**EMPTY-4.** Eval and replay admit explicit empty sparse inputs and apply
these rules in sequence. Eval keeps the original dense input horizon,
including positions whose application causes no tick. Record captures only
actual ticks; dense output has `_` at silent positions. An empty input payload
is present data, not `_` or null. Existing admission diagnostics reject invalid
nonempty input instructions before the graph starts.

Empty atomic complete snapshots and empty ordinary rolling arrivals remain
present publications, including equal repeats. Scalar publication is unchanged.
This contract does not extend [ordinary structural-value publication](structural-value-publication.md)
to empty or wholly invalid held values.

See [reasoned cases](../../../runtime/cases_empty_delta_validity.md) and
[HGL examples](../../examples/empty-delta-validity.hgl).
