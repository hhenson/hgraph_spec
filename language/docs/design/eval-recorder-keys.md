# Eval recorder-key ownership

Status: proposed eval construction rule, 2026-10-03.

Eval owns the keys it supplies to its output recorders. Choosing distinct
keys for those recorders is insufficient if a target or seed already uses
one of the same strings. Before any node starts, eval must choose each
recorder key fresh against the run's closed set of other resolved keys.
This completes key selection for the
[eval composition contract](eval-operator-composition.md); ordinary
[global-state semantics](decisions/0016-eval-scalar-buffer-capabilities.md)
remain unchanged.

## Closed-key preparation

During the existing graph-construction and typed-entry preparation phase,
resolve the source and supplied configuration keys before selecting eval's
recorder keys. The exclusion set contains:

- every resolved source key requirement for the run, whether used by get,
  set, or an independently called recorder, and regardless of bound type;
- every supplied seed key, including a seed not otherwise used by the target;
- every other internal key already selected or required by the run, including
  other eval-owned recording keys.

Known nested graph requirements are included whenever their entries must be
prepared under the existing closed-key profile. A requirement is not omitted
because its hook or branch has not run, a get may never execute, or a nested
graph will be instantiated later. Keys depending on unknown runtime child
data remain outside that profile; this extension does not admit them.

For each eval-owned recording, select an ordinary string unequal to every
key in the exclusion set and add it to that set before selecting another.
Supply the selected string as record's ordinary const key. Bind the entry to
the exact ordinary recording type under the existing prepared-entry rules
before any node starts. A type conflict between user-selected keys remains
an error; choosing fresh internal keys must not rename or repair those
source requirements.

The selection algorithm and key spelling are not part of the eval source
contract. There is no reserved prefix, hidden namespace, new key type or
source-visible entry handle. Freshness is tested by ordinary string equality
over this resolved run, not by an assumption that a generated name is unlikely
to collide. Separate runs retain separate stores and need not select
different strings from one another.

## Runtime behavior and limits

Source-selected keys keep their values, types and ordinary access semantics.
Two source uses of one key still refer to the same entry. Independent record
calls still receive the caller's const string key; this rule protects eval's
generated recording configuration and introduces no store-wide single-writer
restriction or recorder role check.

Start, evaluation and stop use the already prepared typed entries. They do
not discover names, compare source keys against recorder keys, search a
registry or perform runtime type dispatch to maintain this separation.
Recorder start initializes its own typed empty list; admitted publications
append retained captures; extraction after stop reads that same entry.
Source writes to other entries cannot replace or append to this recording
merely by choosing a string that a generator would otherwise have selected.

Existing construction, initialization, evaluation and stop failure contracts
remain in force. Fresh-key preparation completes before start; it adds no
rollback, transaction, in-run rename or fallback successful result. The
existing missing-recording error is not converted to silence.

[Cases](../../../runtime/cases_eval_recorder_keys.md) state the expected
construction and isolation observations. The immutable
[collision audit](https://github.com/hhenson/hgraph_spec_audit/blob/f6e01c1e38f3aa2273d2e0ff94429fd00d9e4bc5/runtime/validation/recorder_keys/README.md)
shows why same-type binding alone cannot establish recorder ownership.
Fresh-key selection is this HGL rule, not a claim of an existing measured
collision-free allocator.
