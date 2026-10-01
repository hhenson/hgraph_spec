# ADR 0016: run-wide keyed state and eval configuration

Status: proposed foundation extension, 2026-09-30.

This promotes the run-wide shared-state concept to a reusable `global_state`
injectable. It supplies ordinary keyed value storage to any requesting node,
independent of that node's inputs, result, name or role. It does not introduce
replay-specific or recording-specific injectables. The
[eval composition contract](../eval-operator-composition.md) remains in force;
complete HGL replay and record bodies require the source contracts listed
below.

## Run ownership and access

A run owns one string-keyed store. Every graph nested within that run shares
it. Separate runs own separate stores even when they instantiate the same
graph description or use identical keys. A run owner may seed ordinary typed
values before graph start and read the final store after all graph stop hooks
have completed. The store lives through that read; an independently owned
result may outlive the run. This is a lifecycle contract, not a new HGL
syntax for constructing or running graphs.

A runtime function requests `inject global_state`. Access is available in
start, evaluation and stop, borrowed for the current hook under INJ-3. The
request adds no signature parameter. Provision one shared store for the run
before any node starts; an unprovisioned request fails graph construction.
There is no fallback to a process-wide store. Nested graphs do not copy or
shadow the store. Independent runs do not inherit earlier entries implicitly.

Keys are ordinary `str` values, compared by their ordinary value equality.
The store assigns no meaning to a key's spelling. There are no reserved
replay/record keys, per-node namespaces, source/sink roles, or single-writer
restrictions. Normal graph execution orders accesses; this extension adds no
concurrent access, transaction or atomic read-modify-write construct.

## Receiver-first operations

These are approved capability calls, using
[receiver-first syntax](../capability-function-syntax.md). V in this table is
specification notation for an ordinary concrete value type, not a new source
type or explicit generic-call syntax.

| Form | Phase | Contract |
|---|---|---|
| `get(global_state, key: str)` with expected result type V | start, evaluation, stop | Read the value stored at key with exactly type V. A missing key or different stored type is an error. |
| `set(global_state, key: str, value: V)` | start, evaluation, stop | Store a value of type V at key under its ordinary ownership contract, inserting or replacing that entry. No result value. |

The checker obtains the expected type of `get` from ordinary value context,
for example `let count: i64 = get(global_state, "count")`. The context must
determine one concrete type. An unconstrained read is a checking error;
there is no type inferred from the key, the runtime contents, or an enclosing
replay/record node's temporal shape. Types must match exactly, including
nominal identity and applicable container shape; get performs no conversion.
A set infers V from its ordinary value argument. Replacement may change an
entry's type; a later read with the earlier expected type then fails.

Stored values are ordinary typed values with an admitted owned value
representation. Temporal endpoints, references to endpoints, injectable
capabilities and borrowed hook views cannot be stored. A contextual
`delta_of(T)` relationship is not by itself an ordinary storable type. This
extension does not make nullable locals or structural delta expressions
storable merely because a key can be chosen for them.

Borrowed store access is bounded to the current hook. The capability cannot
be retained in state/cache, returned from the hook, captured in a closure or
stored in an entry. Reads of `bool`, `i64`, `f64`, `str`, `date`, `time`,
`datetime` and `duration` produce ordinary owned scalar values under their
existing copy rules. A later set at the same key cannot change such a value.

An aggregate get requires a separately admitted value-access contract. This
foundation does not decide whether such a read copies or borrows, how a
mutable aggregate is updated, or how replacement interacts with an earlier
aggregate view. Any eventual borrowed view is bounded to its hook and cannot
be retained as a capture. Until its aggregate access contract is specified,
a source program cannot obtain such a view through get. The store's general
value domain is not implicit authorization for an unspecified aggregate
source operation.

Set retains its argument under the ordinary value type's ownership contract.
The scalar types above use their existing value-copy rules; later changes
to a scalar source do not change the stored value. The store does not promise
a recursive snapshot of every aggregate. Aggregate aliasing, copying,
mutation and replacement lifetimes require that aggregate's admitted
ownership contract before source get/set can use it. Independently owned
retained captures remain a separate obligation of record (VAL-17, EVAL-3),
not an automatic effect of putting any value in shared storage.

A failed set leaves the preceding entry unchanged; it does not undo earlier
successful operations. The run owner must obtain an independently owned
result before disposing of storage on which that result would otherwise
depend. This record does not invent an aggregate copying operation.

Missing-key and wrong-type reads are distinct failures, never `null`, a
default value, or an empty collection. Within hooks, fallible operations use
the [translated node error contract](0009-native-errors-and-the-node-error-model.md):
start failure, evaluation failure and stop failure follow their existing
lifecycle rules. Diagnostics identify the key and distinguish absence from a
type mismatch. Allocation/copy failures propagate without partial entry
replacement. Owner reads after stop likewise fail on missing or wrong-type
entries; they do not manufacture a successful empty result.

Neither operation schedules a node, publishes an endpoint, interprets a
delta, validates a timestamp, deduplicates data, appends to a sequence, or
pads a dense result. Such actions belong to callers and their ordinary value
operations.

For example, a node hook may update an ordinary count whose i64 entry was
seeded by the run owner:

```hgl
fn count_ticks(value: i64) {
    inject global_state
    when {
        let count: i64 = get(global_state, "count")
        set(global_state, "count", count + 1)
    }
}
```

The store neither knows that this entry is a count nor derives its type from
the temporal input. Two nodes choosing the same key see each other's earlier
successful writes in execution order. This source checking is independent of
eval: any runtime function with the requested store may perform the same
scalar calls in its start, evaluation or stop hook.

| Scalar use | Result |
|---|---|
| Read an i64 entry with `let count: i64 = get(global_state, "count")` | An ordinary owned i64. |
| Read stored `false`, zero or empty text with the matching expected type | The present scalar value, never absence. |
| Read with no context determining a concrete result type | Checking error. |
| Read a missing key | Missing-key error. |
| Read an i64 entry in a bool context | Type-mismatch error, no conversion. |
| Replace an i64 entry with a str, then read in an i64 context | Type-mismatch error. |

## Eval, replay and record

The test author still writes `eval(pass_through, value: [...])`. Eval
constructs normal replay source operators, the selected target and a normal
record sink. The generic target remains:

```hgl
fn pass_through<T>(value: T) -> T {
    when { return delta_value(value) }
}
```

Replay receives its finite input sequence directly as an ordinary const
argument. It does not require a global-state key or a storage injectable.
Its exact parameter type depends on the ordinary sequence contracts still
open below; no special replay-data type is introduced to bypass them.
Replay's output type is the corresponding temporal target parameter type.
Const data access does not itself schedule or publish; replay must implement
its cursor, alarm and output behavior.

Record receives the target result and an ordinary const string key:

```hgl
operator record<T>(ts: T, const key: str)
```

This signature states the operator's shape, not universal type admission.
The admitted scalar and collection publication profiles remain specified by
[delta-value metadata](../delta-value-metadata.md) and
[collection eval](../eval-collection-deltas.md). Its implementation requests
`global_state`; the store does not learn a value type from T. Record must
construct an ordinary typed recording value and use a corresponding typed
get/set context. Its recording exists from recorder start even if no tick
arrives (OP-11, EVAL-4). Captured deltas are independently owned (VAL-17,
EVAL-3). The precise ordinary container operations needed to implement this
remain separate source contracts.

Eval owns the recorder's key, replay configuration and any initial values
needed by its graph. It retrieves the recording with its expected ordinary
value type after stop. Its callers do not select keys or configure operators.
Eval chooses distinct recording keys when it constructs distinct recordings
within one run; that is eval's responsibility, not a store writer restriction.
An independent caller of record may supply its own key under that operator's
contract. A missing recording remains an error, not successful silence.

Dense input horizons, repeated equal publications and empty/all-silent
results keep the rules of eval composition. In-range absent sequence elements
are distinct from missing store entries: the former yield `null` under the
[nullable indexing rules](../nullable-replay-indexing.md); the latter fail.

## Unresolved source contracts

These dependencies must be specified before claiming complete executable HGL
replay/record bodies. The keyed store supplies none of them implicitly:

1. **Present/absent sequence elements.** Ordinary `list<value_type>` values,
   homogeneous list literals and const value parameters already exist. What
   remains open is their element representation and construction when a slot
   may be absent, including nullable indexing's relationship to that source
   element type. Eval's harness `_` and local refinement rules do not provide
   that ordinary type or constructor. Admission of nullable element reads
   during wiring or const evaluation also remains to be specified; the
   current local refinement rules do not grant it implicitly.
2. **Ordinary structural delta storage.** Represent recursive sparse deltas
   as ordinary container elements without equating them with held snapshots.
   `delta_of(T)` remains a specification-only relationship, not an annotation.
3. **Owned recording values.** Specify ordinary empty construction,
   timestamp/delta entry representation, growth or functional replacement,
   and extraction. This includes ownership of nested data; a borrowed
   endpoint view is not a retained capture. No recording-specific append or
   begin primitive substitutes for these general value operations.
4. **Generic source checking.** Express the relationship between replay's
   const sequence element type and temporal result, and between a record
   input and its ordinary recording value. No injectable silently supplies
   a missing shape constraint or inference rule.
5. **Aggregate access and retention.** Specify ordinary get/set ownership,
   aliasing, mutable access, replacement lifetime and any copying needed for
   aggregate values. Hook-bounded borrowed access cannot stand in for an
   independently owned retained recording. Only the scalar get/set forms
   above are fully defined here.
6. **Eval key selection.** Distinct recorder keys alone do not prevent target
   code from choosing the same ordinary string key. Eval's collision policy
   with user-chosen entries remains to be specified. This foundation does
   not reserve a hidden namespace or promise collision-free access through
   node roles; general store replacement retains its ordinary meaning.
7. **Replay admission and bounds.** Ordinary const-sequence admission must
   place finite length/index representability and dense slot-time validation
   at a defined checking or construction boundary. ENG-3/ENG-16 still bound
   every executed instant; generic get/set neither checks nor changes those
   time rules.

Persistence, checkpoint/restart, shared state across separate runs and traits
remain outside this foundation. It settles a reusable store and eval's
configuration direction, not a complete general resource language.
