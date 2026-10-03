# ADR 0016: run-wide keyed state and eval configuration

Status: proposed foundation extension, corrected typed-access profile 2026-10-01.

This promotes the run-wide shared-state concept to a reusable `global_state`
injectable. It supplies ordinary keyed value storage to any requesting node,
independent of that node's inputs, result, name or role. It does not introduce
replay-specific or recording-specific injectables. The
[eval composition contract](../eval-operator-composition.md) remains in force;
the [ordinary scalar data contract](../../../../library/ordinary_replay_record.md)
supplies replay and record bodies for eight scalar types, generalized to the
admitted structural profile by [ordinary delta types](../ordinary-delta-types.md).
The remaining source boundaries are listed below.

## Run ownership and access

A run owns one string-keyed store. Every graph nested within that run shares
it. Separate runs own separate stores even when they instantiate the same
graph description or use identical keys. A run owner may seed ordinary typed
values before graph start and read the final store after all graph stop hooks
have completed. The store lives through that read; an independently owned
result may outlive the run. This is a lifecycle contract, not a new HGL
syntax for constructing or running graphs.

Eval narrows this general pre-start window: its owner-supplied seeds must be
finalized before eval selects internal recorder keys. Later seed additions or
replacements for that eval are configuration errors before start, under
[eval recorder-key ownership](../eval-recorder-keys.md). This does not change
the seeding window for other run owners or ordinary node-hook writes.

A runtime function requests `inject global_state`. Access is available in
start, evaluation and stop, borrowed for the current hook under INJ-3. The
request adds no signature parameter. Provision one shared store for the run
before any node starts; an unprovisioned request fails graph construction.
There is no fallback to a process-wide store. Nested graphs do not copy or
shadow the store. Independent runs do not inherit earlier entries implicitly.

Keys are ordinary `str` values, compared by their ordinary value equality.
In this source profile, the key argument must be a const expression resolved
before the run starts. String literals and ordinary const string parameters
are admitted; a key depending on a temporal input or a mutable hook local is
not. This uses existing const-expression checking, not a new key type.

The store assigns no meaning to a key's spelling. There are no reserved
replay/record keys, per-node namespaces, source/sink roles, or single-writer
restrictions. Normal graph execution orders accesses; this extension adds no
concurrent access, transaction or atomic read-modify-write construct.

## Prepared typed entries

Each used key is associated with one exact ordinary value type for the run.
The type is concrete after generic specialization. Get obtains it from its
ordinary expected value context; set obtains it from its ordinary argument.
Neither derives a type from a node role or a key's spelling.

Before any node starts, resolve the const keys, reconcile their type
requirements, validate supplied seed values, and prepare each node's access
to the corresponding run-owned typed entry. Equal keys and equal types in
one run refer to the same entry, including across nested graphs. Different
runs have independent entries and may use the same key with different types.
An entry's bound type cannot change during that run.

Known incompatible requirements for the same key within a function or a
statically assembled run are checking errors. If separately supplied const
key configurations resolve to the same string with incompatible types,
graph construction fails before any node starts. A supplied seed whose type
differs from the bound type also fails construction. Types match exactly,
including nominal identity and applicable ordinary container shape; there
is no conversion during binding. Distinct entries with conflicting types
must not be allocated under the same key to avoid a conflict.

During start, evaluation and stop, a get/set uses its prepared typed entry.
It performs no string-key lookup, registry search, runtime type dispatch or
node-role inspection. This does not promise allocation-free scalar copying:
ordinary `str` ownership rules still apply. Entry bindings are not source
values, explicit handles, capabilities to store elsewhere, or a new argument
on the node's callable signature.

Preparing an entry does not give it a value. An unseeded entry is absent
until a successful set initializes it. Required get tests this presence and
fails when absent; it never manufactures zero, false, empty text or null.
A start hook may initialize such an entry for later reads. A get that is not
executed need not have an initialized value. Type binding is unconditional;
value presence follows executed writes.

The current profile admits only keys and type requirements resolved before
root graph start. A later-created nested graph may use already prepared
entries; introducing a key that depends on runtime child data is outside
this profile. This does not change graph scheduling or create a per-role
storage service.

## Receiver-first operations

These are approved capability calls, using
[receiver-first syntax](../capability-function-syntax.md). V in this table is
specification notation for an ordinary concrete value type, not a new source
type or explicit generic-call syntax.

| Form | Phase | Contract |
|---|---|---|
| `get(global_state, key: str)` with const key and expected result type V | start, evaluation, stop | Read the prepared V entry. An absent value is an error; its type was validated before start. |
| `set(global_state, key: str, value: V)` with const key | start, evaluation, stop | Initialize or replace the value in the prepared V entry under its ordinary ownership contract. Its bound type is unchanged. No result value. |

The checker obtains the expected type of `get` from ordinary value context,
for example `let count: i64 = get(global_state, "count")`. The context must
determine one concrete type. An unconstrained read is a checking error;
there is no type inferred from the key, the runtime contents, or an enclosing
replay/record node's temporal shape. Types must match exactly, including
nominal identity and applicable container shape; get performs no conversion.
A set infers V from its ordinary value argument. Every use of the same key
must agree with that exact type. A type-changing set is rejected during
checking or construction, not deferred to a later get. Dynamic runtime keys
and in-run changes to an entry’s bound type are outside this profile.

Stored values are ordinary typed values with an admitted owned value
representation. Temporal endpoints, references to endpoints, injectable
capabilities and borrowed hook views cannot be stored. The separate
[ordinary delta type contract](../ordinary-delta-types.md) admits `delta<T>`
for its finite profile and requires independent retention. The store does
not infer this type from a node role or make nullable locals storable merely
because a key can be chosen for them.

Borrowed store access is bounded to the current hook. The capability cannot
be retained in state/cache, returned from the hook, captured in a closure or
stored in an entry. Reads of `bool`, `i64`, `f64`, `str`, `date`, `time`,
`datetime` and `duration` produce ordinary owned scalar values under their
existing copy rules. A later set at the same key cannot change such a value.

[Value mutability](../value-mutability.md) supplies the aggregate access
contract. A typed `let` initialized by get borrows read-only access; a typed
`var` borrows exclusive writable access to the existing entry, without a
whole-value copy. The borrow ends at its lexical block or hook. Entries are
mutable storage, regardless of the access used to read them. Access mode is
not part of their exact bound type. Conflicting same-key get/set lifetimes
fail checking or binding before start; get adds no type or key dispatch.
This access contract does not supply missing construction or list operations.

Set retains its argument under the ordinary value type's ownership contract.
The scalar types above use their existing value-copy rules; later changes
to a scalar source do not change the stored value. The store does not promise
a recursive snapshot merely because an aggregate is read. Under the
value-mutability contract, aggregate get borrows, while an admitted retention operation
stores an independent owning value rather than a view handle. Set's value
argument is such an owning retention boundary for admitted ordinary values;
it cannot replace an entry with a live conflicting borrow. Independently
owned retained captures remain a separate obligation of record (VAL-17,
EVAL-3), not an automatic consequence of obtaining mutable access.

A failed set leaves the preceding entry unchanged; it does not undo earlier
successful operations. The run owner must obtain an independently owned
result before disposing of storage on which that result would otherwise
depend. This record does not invent an aggregate copying operation.

Missing values and incompatible types remain distinct errors. Type conflicts
are diagnosed during checking or construction as described above; there is
no wrong-type alternative after a successful typed binding. An executed get
of an uninitialized entry fails under the
[translated node error contract](0009-native-errors-and-the-node-error-model.md).
Start failure, evaluation failure and stop failure follow their existing
lifecycle rules. Diagnostics identify the key and distinguish missing value
from type conflict. Allocation/copy failures propagate without partial entry
replacement. Owner reads after stop must request the entry's bound type and
fail on missing values or incompatible requested types; they do not
manufacture a successful empty result. Owner configuration/extraction may
resolve keys and validate types outside hooks; hook access uses only the
prepared typed entry.

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
| Bind an i64 seed to a bool read | Construction error before any node starts. |
| Set an i64 and a str at the same known key | Checking error; no type-changing write. |
| Distinct const key parameters resolve to one key with incompatible types | Construction error before any node starts. |
| Use a temporal string input as key | Checking error: key is not a const expression. |

## Source acceptance and rejection

This reusable sink initializes an ordinary counter in start, increments it
for admitted ticks and leaves its final value in the shared entry at stop.
A caller may configure any const string key. The start write initializes the
value; merely binding the key does not.

```hgl
fn counter(value: i64, const key: str) {
    inject global_state
    start { set(global_state, key, 0) }
    when {
        let count: i64 = get(global_state, key)
        set(global_state, key, count + 1)
    }
    stop {
        let count: i64 = get(global_state, key)
        set(global_state, key, count)
    }
}
```

The key and i64 type are prepared once, independently of whether the node is
used under eval or an ordinary entry graph. Reading before that first set
would fail if no seed supplied a value. Reading stored false, zero or empty
text succeeds in its exact scalar context.

The following fragments are checking failures:

```hgl
fn dynamic_key(key: str) {
    inject global_state
    when { set(global_state, key, 1) }
}

fn changing_type(value: i64) {
    inject global_state
    when {
        set(global_state, "counter", 1)
        set(global_state, "counter", "one")
    }
}
```

The first key is temporal, not const. The second function imposes incompatible
i64 and str requirements on one key. A same-type replacement is allowed.
Separately valid counter functions with different configured keys do not
conflict merely because they use the same generic body. Bind-time conflicts
are scoped to a single run, not to all functions declared in a module.

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
For the eight scalar types its exact parameter type is ordinary
`list<TimedValue<T>>`, specified by the ordinary scalar data contract. There
is no absent element in that list; no special replay-data type is introduced.
For the admitted structural profile, the ordinary delta type extension uses
`list<TimedValue<delta<T>>>`; scalar reduction preserves the same data type.
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
EVAL-3). Ordinary construction and retained list push use the explicit type
`list<TimedValue<delta<T>>>` for the admitted delta-type profile. The store
adds no publication admission or delta inspection operation.

Eval owns the recorder's key, replay configuration and any initial values
needed by its graph. It retrieves the recording with its expected ordinary
value type after stop. Its callers do not select keys or configure operators.
Eval chooses distinct recording keys when it constructs distinct recordings
within one run, fresh against all resolved source, seed and other internal
keys under [eval recorder-key ownership](../eval-recorder-keys.md). This is
eval's construction responsibility, not a store writer restriction.
An independent caller of record may supply its own key under that operator's
contract. A missing recording remains an error, not successful silence.

Dense input horizons, repeated equal publications and empty/all-silent
results keep the rules of eval composition. Scalar replay does not need an
ordinary nullable sequence: eval keeps absent slots in its harness metadata
and supplies only present timed entries. Where separately admitted, in-range
absent sequence elements are distinct from missing store entries: the former
yield `null` under the [nullable indexing rules](../nullable-replay-indexing.md);
the latter fail.

## Unresolved source contracts

The ordinary list, ownership and delta type contracts supply the bounded
replay/record data profile. Remaining boundaries, and where dependencies
were resolved, are listed here; the keyed store supplies none implicitly:

1. **Present/absent sequence elements.** Ordinary `list<value_type>` values,
   homogeneous list literals and const value parameters already exist. What
   remains open is their element representation and construction when a slot
   may be absent, including nullable indexing's relationship to that source
   element type. Eval's harness `_` and local refinement rules do not provide
   that ordinary type or constructor. Admission of nullable element reads
   during wiring or const evaluation also remains to be specified; the
   current local refinement rules do not grant it implicitly. Ordinary timed
   replay avoids this dependency by omitting absent slots from ordinary data.
2. **Ordinary structural delta storage.** Supplied for the finite profile by
   [ordinary delta types](../ordinary-delta-types.md), using source `delta<T>`
   without equating sparse publication data with held snapshots. Excluded
   publication semantics remain separate from data formation and retention.
3. **Owned recording values.** The [ordinary-list extension](../ordinary-list-values.md)
   supplies typed empty construction, length, indexed extraction and retained
   end growth. TimedValue with a `delta<T>` payload supplies both the admitted
   scalar and structural recording entries.
   Ordinary owning retention includes nested data; a borrowed endpoint view
   is not a retained capture. No recording-specific append or begin primitive
   substitutes for these general value operations.
4. **Generic structural source checking.** Supplied by `delta<T>` formation
   and exact originating-shape matching. These relate the ordinary timed
   list to temporal T and the recording type. No injectable silently supplies
   this type relationship or infers it from payload contents.
5. **Aggregate operations.** The value-mutability contract defines binding
   authority, borrowed entry access, conflicting lifetimes and independent owning
   retention. The ordinary-list extension supplies a focused construction/read/growth
   contract. Other aggregate operations and any explicit general copying
   operation still require their value contracts. A borrowed aggregate is not
   an independently owned retained recording.
6. **Eval key selection.** Supplied by
   [eval recorder-key ownership](../eval-recorder-keys.md): before start,
   finalize eval's owner-supplied seed configuration, then choose each
   eval-owned recording key fresh against the closed resolved
   source/seed/internal key set, including known nested requirements. No
   hidden namespace or node-role check is added; source-selected same-key
   access and ordinary replacement retain their meanings.
7. **Eval admission and bounds.** Ordinary list operations
   supply length/index representability. Dense slot-time normalization still
   needs its checking or construction boundary. Independent scalar replay
   uses the existing generator rules, including skipped past entries and
   duplicate-time errors; it does not validate strictly increasing times
   before start. ENG-3/ENG-16 still bound every executed instant; generic
   get/set neither checks nor changes those time rules.

Persistence, checkpoint/restart, shared state across separate runs and traits
remain outside this foundation. It settles a reusable store and eval's
configuration direction, not a complete general resource language.
