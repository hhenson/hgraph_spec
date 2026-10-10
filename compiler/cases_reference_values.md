# Graph-wired reference observations

These cases use [REF authoring](../language/docs/design/type-extensions.md#node-access-and-ticks),
[routing](../language/examples/reference-routing.hgl), WIR-6–14 and
TS-14/16/17/20/22/23/25. Inputs to eval are ordinary scalar or structural
publication traces. Graph bindings obtain references; scalar or complete
atomic observation outputs expose results. No raw REF input trace, REF delta
constructor or empty-reference literal is admitted by these cases.

| Case | Required observation |
|---|---|
| Ordinary i64 output binds to `ref<i64>`, is forwarded, then binds to an ordinary i64 consumer | Follow target publications, including equal ticks and silence; the routing node need not copy later target data. |
| Compare references obtained from two bindings to the same endpoint, including through a REF relay | Equal while readable, regardless of target-value changes. |
| Compare two different endpoints holding the same scalar values | Unequal; endpoint identity does not collapse to payload equality. |
| Select A, repeat A, switch to B, repeat B, return to A | First/changed designations tick; repeats do not tick or sample. Valid changed targets are sampled at the rebind cycle. |
| A and B hold equal values when switching | The changed designation still ticks and the ordinary follower samples B. |
| Target publishes while selector is silent | Data follower receives the publication; REF modification/tick count does not change. |
| Observer step runs before selector's first publication | Routed REF and following data input are invalid and unmodified; do not read either payload. |
| Select a bound target that has never published | The published designation is valid; the ordinary follower is invalid and unmodified until its target publishes. |
| Repeat that invalid target's designation | No new REF tick and no sample; later target publication makes the follower valid without a selector tick. |
| Rebind to a valid target whose last publication was earlier | Following input's observed time becomes the current sample time; producer time stays unchanged. |
| A REF output publishes its first incoming designation once and ignores later incoming changes | It holds that designation across cycles; the ordinary follower receives later ticks from the original live target. It holds a connection rather than a copied scalar value. |
| A fresh run captures a different first designation | No designation leaks across graph teardown/runs. |
| Project an ordinary fixed-list child or nominal field before binding it to a REF input | Preserve child identity, initial invalidity, equal ticks and sibling silence under WIR-5. |

## Boundaries requiring separate work

Graph access below REF follows WIR-5 and the accepted type-extension contract;
its separate [projection clarification](../language/docs/design/type-extensions.md#wiring-time-access)
has its own examples/cases. Node access below REF remains opaque.

The runtime's REF-SAMPLE empty/unbind row and REF-EXPIRES (TS-23) remain
requirements in [reference cases](../runtime/cases_references.md). HGL has no
empty-reference constructor. Current state/cache annotations use `value_type`,
which excludes `ref`, and global value storage does not admit temporal
endpoints. A REF output can hold a designation without those storage forms,
but these graph-wired fixed/scalar cases do not dispose of an ephemeral
dictionary child. They therefore do not establish expiry, generation/key reuse,
cross-graph reference escape or saved-reference ordinary storage. Dynamic-child
capture and expired-designation fixtures need a separately specified source
path; no new lifecycle or storage surface is inferred here.
