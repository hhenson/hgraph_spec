# Conditional reference expiry cases

Required contract for the narrow stopped-owner clarification of
[TS-23](../runtime/time_series.md), together with
[GRF-10/19/20/23/25](../runtime/graph.md) and
[conditional branch ownership](../language/docs/design/control-flow.md#branch-ownership).
This case document claims no HGL target execution.

A temporal conditional composes a scalar producer inside each branch and
returns its designation through an explicitly `ref<i64>` result. The parent
captures the first designation in a REF output with a scalar Boolean latch;
no reference is placed in ordinary state. An independently ticking observer
binds that held output and the current conditional result to ordinary scalar
inputs. All routes cross the existing child-graph output boundary. Stop affects
the branch-owned producer, not the captured outer source.

| Cycle | Enabled | First source | Second source | Held value | Current value |
|---|---|---|---|---|---|
| 1 | true | 7 | 20 | 7 | 7 |
| 2 | true | 8 | silence | 8 | 8 |
| 3 | false | 9 | silence | 8 | 20 |
| 4 | false | 10 | 21 | absent | 21 |
| 5 | true | 11 | silence | absent | 11 |
| 6 | true | 12 | silence | absent | 12 |
| 7 | false | 13 | 22 | absent | 22 |
| 8 | true | 14 | silence | absent | 14 |
| 9 | silence | 15 | 23 | absent | 15 |

The held input is valid/modified at cycles 1 and 2, valid/unmodified at cycle
3, and invalid/unmodified from cycle 4. The current input is valid/modified
at every listed cycle. At cycle 3, the stopped producer retains 8 through the
cycle; its now-unbound source input does not process 9. At cycle 4, logical
expiry is observable regardless of physical reclamation. Fresh true-branch
instances at cycles 5 and 8 preserve current routing without reviving the
saved designation. A second eval repeats the trace with fresh parent state.

Boundary requirement: the explicit REF result must expose
the branch-owned intermediate scalar designation through GRF-10, rather than
substituting a stable parent scalar endpoint. The existing REF result-schema
rule in control-flow preserves that designation; the fixture deliberately
puts the scalar producer before the final REF forwarding node. Metadata above
concerns ordinary followed scalar inputs, not the retained REF output's own
validity/timestamp after its designation expires. It asserts no expiry tick
or teardown notification.
