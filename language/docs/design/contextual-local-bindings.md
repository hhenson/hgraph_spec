# Contextual local bindings

`let` and `var` do not select scalar or time-series representation. The
expression and its context do. A binding has one canonical type and one
category: an ordinary value or a time-series connection.

- In node execution, locals hold ordinary values or the ordinary observations
  explicitly permitted by the access rules. Collections and structs are also
  ordinary values; reading an input does not make the local a connection.
- In graph composition, an ordinary initializer creates an ordinary binding;
  a temporal initializer creates a connection binding. A type annotation
  checks the type in that category; it does not lift a value into a stream.
- Initialization fixes both type and category. `let` forbids reassignment.
  `var` permits replacement within that fixed type and category. Existing
  ordinary numeric widening remains applicable; it does not change category.
  Scalar-to-connection and connection-to-scalar assignment are type errors,
  even when both involve the same leaf type. Check unused assignments too.
  For compound assignment, check the computed result: a connection `x += 1`
  may wire addition and rebind x to its compatible temporal result.
- Rebinding a graph connection changes the local name during wiring. Earlier
  aliases and connections already consumed by other calls keep their original
  targets. It neither mutates an upstream value nor creates runtime state.

`atomic<V>` controls a temporal publication boundary, not local mutability.
An ordinary composite local uses V; its annotation cannot create an atomic
endpoint. Existing non-composite atomic normalization remains unchanged.
Ownership, read-only access and lexical borrows follow
[value mutability](value-mutability.md).

A typed `var` without an initializer supplies no value. Its category must be
fixed by its first assignment outside a temporal-result context, and must be
resolved consistently under the existing
[definite-assignment and conditional-result rules](control-flow.md#results-used-after-the-conditional),
not chosen by the order in which branches are checked. All assignments to one
binding must agree after applying their permitted contextual rules. In
particular, an uninitialized escaping result of a temporal conditional has a
temporal branch-output context: scalar branch results may be lifted under the
existing conditional contract. Resolve that slot as temporal before checking
its branches; assignments within those branches, including repeated ones,
supply temporal branch outputs, and reads observe connections. After the
conditional, ordinary fixed-category assignment rules apply. This is initial
construction of a connection,
not permission to change an initialized local. An initialized ordinary local
cannot escape as a temporal conditional result; an initialized connection
cannot be assigned an ordinary scalar inside a branch either. Existing scalar
lifting at call and return boundaries remains unchanged. A temporal branch
cannot write an initialized ordinary binding from an enclosing scope, even
if nothing later reads it. Ordinary locals declared within the branch remain
ordinary; a wiring-time conditional may update enclosing ordinary `var` values.

See [examples](../../examples/contextual-local-bindings.hgl) and
[compiler cases](../../../compiler/cases_contextual_local_bindings.md).
