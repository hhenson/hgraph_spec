# Map key connection cases

Required expectations under [MAP-CONN-1–4](../language/docs/design/map-key-connections.md),
TS-11/14/16/17/23 and [LOOKUP-1–4](../language/docs/design/nullable-map-lookup.md). The [source examples](../language/examples/map-key-connections.hgl)
use ordinary map publication deltas and graph-wired references. `_` in an
output trace is no publication; an empty REF is a valid designation value.
This document claims no target execution. The initial V profile has no REF
leaves, including below an outer `ref<map<K,V>>`.

Observe after producers and selection evaluate. Select X initially; A is its
first child, B its later replacement, and C the Y child. Hold A in a separate
REF output when it first becomes readable. The saved ordinary follower is
observed by an independent step input; the selector's ordinary follower is
observed for publications.

| Cycle | Map change | Key publication | Selection / REF tick | Ordinary publication | Saved A value |
|---|---|---|---|---|---|
| 0 | Publish empty map | X | empty / yes | _ | absent |
| 1 | Insert X=7 | _ | A / yes | 7 | 7 |
| 2 | Insert Y=20 | _ | A / no | _ | 7 |
| 3 | X=8 | _ | A / no | 8 | 8 |
| 4 | Remove X | _ | empty / yes | _ | 8 |
| 5 | Publish empty delta | _ | empty / no | _ | absent |
| 6 | Insert X=8 | _ | B / yes | 8 | absent |
| 7 | None | Y | C / yes | 20 | absent |
| 8 | None | Y | C / no | _ | absent |
| 9 | None | Missing | empty / yes | _ | absent |
| 10 | None | OtherMissing | empty / no | _ | absent |
| 11 | Y=21 | _ | empty / no | _ | absent |

At 0 the REF is valid/modified while the ordinary follower is invalid and
unmodified. At 4 selection has already cleared; only the independently held A
designation still reads 8. At 6 equal payloads do not equate A and B: B is
sampled, and the held A remains expired. At 7 C's held 20 is sampled without a
new Y publication; its producer timestamp is unchanged. Key/membership changes
maintain selection without requiring the key to republish at 1, 4 or 6.

| Additional case | Required result |
|---|---|
| Valid key before the map's first publication | Publish a valid, modified empty REF; the ordinary follower is invalid/unmodified. Repeated absence is silent, and later arrival with silent key selects and samples the child. |
| Select a key through `ref<map<K,V>>` in graph composition | Compose WIR-5 with live selection; parent-map retargeting reselects the child for the held key, including equal-valued targets, missing membership and a silent key. |
| A present selected child is invalid | Membership stays true; designate that child and apply TS-14/17's invalid-target observations. Do not publish empty REF. |
| Selected child becomes invalid, then valid again without removal | Keep membership and designation unchanged; propagate ordinary invalidation/revalidation under existing input rules, without a REF tick. |
| Switch X to a different live Y that holds the same 7 | REF ticks and the ordinary consumer samples 7 again; equal payloads do not equate target identities. A repeated Y key is silent. |
| Remove and restore the key within the same cycle | Retain the same child under TS-11; do not infer a new identity from the later-cycle reinsertion row. |
| Ordinary/atomic `value[key]` with absent key | Lookup fails; no empty REF, fabricated scalar or implicit no-publication result. |
| Ordinary/atomic `get(value, key)` with absent key | Successful `null` result; explicit `default: 0` returns 0. |
| Ordinary/atomic `get(value, key)` with present zero | Present 0, distinct from absence. |

The source fixture checks the temporal trace, equal-payload retargeting and
ordinary/atomic lookup controls, including the existing `value.missing_key`
failures for brackets and `at`. Safe lookup uses guarded local null observations;
it does not return a nullable value through a declared result. Invalid key
observations, whole-map invalidation and safe lookup of present invalid temporal
children remain outside this matrix. Holding A uses a REF output and a scalar latch,
not ordinary state containing a reference.

REF-valued and REF-containing child selection remains a follow-up. Existing
map-of-REF formation/binding is unchanged; its absence from this selection
matrix is not a type-formation rejection.
