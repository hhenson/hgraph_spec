# Temporal scalar publications

Add `civil_datetime`, `timezone`, `zoned_datetime` and `zoned_time` to the finite scalar
publication profile. Its twelve built-in leaves are `bool`, `i64`, `f64`, `str`, `date`,
`time`, `datetime`, `duration`, and these four types. The [enum extension](enum-publications.md) additionally
admits declared enums; other scalar families remain outside this profile.

For each admitted S, `delta<S>` is S and `atomic<S>` normalizes to S. The same
leaf admission applies recursively to structural deltas and finite atomic
payloads. Existing collection restrictions, shape identity and composite
inference rules remain unchanged.

Use the existing guarded `delta_value`, generic pass-through, `TimedValue<S>`,
replay and record contracts. Present equal values still tick; `_` is silence.
Owning capture retains an independent ordinary scalar. Later publications,
another eval or graph teardown cannot alter an earlier result.

Preserve the [existing temporal identities](../developer-guide/syntax-and-semantics.md#temporal-scalar-types):

- Civil values retain their wall-clock fields; they do not become UTC instants.
- Zone values retain the exact name. Zone aliases are not canonicalized.
- Zoned times retain their exact wall-clock time (including microseconds) and
  exact zone name. They carry no date, instant or resolved offset. Equal wall
  times in different zones, including alias spellings, remain distinct values.
- Zoned datetimes retain instant, zone and resolved offset. Same-instant values
  in different zones remain different values under ordinary equality.

Publication, recording and harness comparison use those ordinary identities;
they introduce no conversion or timeline-only equality. Literal syntax and
strict provider validation retain their existing rules. This admission adds
no provider injectable or time-zone resolution operation.

For eval, make its required run context available while materializing
provider-dependent input literals. Evaluate supplied expressions once in their
existing written order; do not defer or reevaluate general inputs. Complete
and validate all supplied values before any target starts. Construction or
strict-decoding failure aborts eval before start. Retaining or replaying an
already constructed valid scalar preserves its exact value without provider
re-resolution. This defines no general provider lifetime or configuration API.

See [examples](../../examples/temporal-scalar-publications.hgl) and
[compiler cases](../../../compiler/cases_temporal_scalar_publications.md).
