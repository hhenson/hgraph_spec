# Scalar collection keys and members

Extend the existing structural publication profile to `set<K>` and `map<K, S>`,
where K is any admitted built-in scalar or declared enum and S is any admitted
publication shape. This includes bytes under [BYTE-1–6](bytes-values.md), with
exact content equality/hash. For `f64`, this extension covers non-NaN values, including
both infinities. NaN membership and key application remain an explicit open
boundary; this record does not choose a NaN equality policy. The separate [finite composite-key extension](composite-collection-keys.md)
admits tuples and concrete structs; [native atomic keys](native-atomic-values.md) are separately admitted by NVAL-5.

Keys and members retain their exact ordinary type and identity. Equality and
hash determine membership; ordering is unnecessary. Equal values must have
equal hashes. Enum identity is not its integer representation; zone aliases
are distinct exact names; zoned datetimes retain instant, zone and offset;
zoned times retain wall-clock time and zone without a date or offset.
`+0.0` and `-0.0` identify the same `f64` key/member. Positive and negative
infinity identify distinct keys/members. No stringification, implicit numeric
conversion, zone canonicalization or cross-type equality is introduced.

Use the existing delta constructors and generic pass-through. In sparse map
`upsert` entries, replace the constant-i64-only key rule with a constant of
exact K. Set `added`/`removed` and map `remove` lists likewise contain exact K
constants. Fixed list/tuple indices remain in-range constant i64 values.
This adds neither dynamic keys nor a general ordinary map/set literal.

An immutable ordinary `let` initialized from an admitted key constant or cold
recipe may name that retained key in a later sparse constructor. Read the
prepared value; do not replay its initializer. Alias chains preserve exact K
and ordinary value retention. This rule does not admit temporal bindings,
mutable bindings or arbitrary node-runtime expressions as sparse keys.

A provider-dependent constant remains a cold input recipe under the existing
temporal materialization contract. Construct and validate it once in written
order before any target starts, using the required run context. Retain its
exact result for replay and publication; perform no per-tick provider lookup.
Reject duplicates and added/removed or upsert/remove overlap using K equality:
when values are available during checking, diagnose there; otherwise diagnose
during cold materialization before start. Do not guess identity from text or
resolve a provider-dependent key during source checking without its context.
Existing type, input-profile and construction-failure diagnostics apply.

Harness comparison ignores set-member and map-entry order while preserving
exact keys and recursive child deltas. State-dependent canonical membership,
removal, ownership, replay and recording rules apply, including
[empty sparse application](empty-delta-validity.md). An equal scalar child
update still publishes at its key. Invalidation, invalid child creation and
references retain their separate boundaries.

See [examples](../../examples/scalar-collection-keys.hgl) and
[compiler cases](../../../compiler/cases_scalar_collection_keys.md).

The [any extension](any-values.md) admits boxed keys with contained equality
and hash; missing capabilities fail at use under ANY-5.
