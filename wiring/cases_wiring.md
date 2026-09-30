# Wiring cases

Status: validated expectations with recorded variations; rewritten for
clarity on 2026-09-30 with the case names, rule references and expected
decisions unchanged. The observations are in the
[wiring validation](https://github.com/hhenson/hgraph_spec_audit/blob/main/runtime/validation/wiring/README.md).

## How to read a case

Each case describes a small graph in words, then says what wiring
**decided** — the type of a port, which candidate a call selected, whether
a call failed — and which rule made it decide so. Where a decision only
shows once the graph runs, the case also gives the values a node sees, per
cycle from the first; `—` means nothing was published that cycle.

Type notation is the runtime's: `TS<i64>` is a series of integers,
`TSL<TS<i64>, 3>` a fixed list of three, `TSD<str, TS<i64>>` a dictionary
keyed by strings, `TSB{a: TS<i64>}` a bundle with one field, `REF<X>` a
reference to a time-series of type `X`, and `T` a type variable. In the
recorded runs these are hgraph's Python spellings (`TS[int]`,
`TSD[str, REF[TS[int]]]`); the correspondence is one to one.

Three helper nodes recur:

| Node | Signature | What it does |
|---|---|---|
| `identity<T>` | `(ts: T) -> T` | publishes its input unchanged: the plainest generic node |
| `to_ref` | `(ts: REF<TS<i64>>) -> REF<TS<i64>>` | publishes a reference to its input |
| `to_map` | `(list: TSL<TS<i64>, 3>) -> TSD<str, REF<TS<i64>>>` | publishes a dictionary with keys `a`, `b`, `c`, each holding a *reference* to one element of the list. This is the shape a keyed nested construct's output has |

## WIRE-GENERIC-DEPTH — WIR-7, WIR-8

*The rule of rules: a generic binds the value type, with every reference
removed, at every depth.*

**Setup.** A source publishes a list of three integers: `(1, 2, 3)` at
cycle 0, then element 1 becomes 30 at cycle 1. `to_map` turns the list into
`TSD<str, REF<TS<i64>>>`. That dictionary is passed to `identity<T>`.

**What wiring decides.**

| Question | Answer |
|---|---|
| The type of `to_map`'s port | `TSD<str, REF<TS<i64>>>` |
| What `T` binds when that port is passed to `identity<T>` | `TSD<str, TS<i64>>`: the references are removed, one level down |
| The type of `identity`'s port | `TSD<str, TS<i64>>` |

**What the graph then shows.** `identity`'s input follows each reference,
so it sees values, not references: at cycle 0 `a=1, b=2, c=3`; at cycle 1
`a=1, b=30, c=3`, and the input reads modified at cycle 1 because a value
behind a reference ticked.

**Why.** WIR-7 removes references at every depth when a variable binds
from an argument; WIR-8 says the generic input therefore observes values.
(hgraph issue #847.)

## WIRE-GENERIC-TOP — WIR-7, WIR-8

*The same rule at the top level.*

**Setup.** A source publishes `1` at cycle 0 and `2` at cycle 1. `to_ref`
publishes a reference to it, and that reference is passed to `identity<T>`.

| Question | Answer |
|---|---|
| The type of `to_ref`'s port | `REF<TS<i64>>` |
| What `T` binds | `TS<i64>` |
| What `identity` sees at cycles 0 and 1 | `1`, then `2` |

## WIRE-REF-PATTERN — WIR-10

*A reference in the pattern is how a node asks for one.*

**Setup.** A node `holds<T>` declares its input as `ts: REF<T>` and its
output as `REF<T>`.

| Call | What `T` binds | The output type |
|---|---|---|
| `holds(to_map(...))`, the dictionary of references above | `TSD<str, TS<i64>>`: beneath the declared reference, then references removed | `REF<TSD<str, TS<i64>>>` |
| `holds(source)`, a plain `TS<i64>` | `TS<i64>` | `REF<TS<i64>>`; the input receives a reference to the source, since a reference and its target are interchangeable at a binding |

**Why.** WIR-10: the variable binds beneath the pattern's reference;
below that, references are removed as for any argument.

## WIRE-STATED — WIR-11

*A caller who states the binding keeps the references.*

**Setup.** `identity<T>` is called with `T` stated before matching.

| Stated `T` | Argument | Result |
|---|---|---|
| `TSD<str, REF<TS<i64>>>` | the dictionary of references | wires; `T` stays as stated, references and all |
| `REF<TS<i64>>` | `to_ref(source)` | wires; `T` stays `REF<TS<i64>>` |

**Why.** WIR-11: a stated resolution is kept, references included, and an
argument matches it as supplied or with its references removed.

## WIRE-REQUESTED — WIR-12

*A requested output type is honoured as written.*

**Setup.** `nothing` is a source whose output pattern is a bare variable;
it publishes nothing and exists so that a caller can ask for an output of
a given type.

| Requested output | The port's type |
|---|---|
| `TSD<str, REF<TS<i64>>>` | `TSD<str, REF<TS<i64>>>` |
| `REF<TS<i64>>` | `REF<TS<i64>>` |
| `TSD<str, TS<i64>>` | `TSD<str, TS<i64>>` |

**Why.** WIR-12: a variable that is the whole output pattern binds the
requested type with its references kept.

## WIRE-PROJECTION — WIR-5, WIR-13

*Selecting a known part of a port adds no node and keeps the declared
type.*

**Setup.** A node publishes a bundle `TSB{routed: REF<TS<i64>>, plain:
TS<i64>}`. A graph selects the `routed` field, by index, by name and by
attribute spelling, and returns it as its own `TS<i64>` output. The source
behind `routed` publishes `1` at cycle 0 and `2` at cycle 1.

| Selection | Type of the selected part |
|---|---|
| `bundle["routed"]` | `REF<TS<i64>>` |
| `bundle.routed` | `REF<TS<i64>>` |
| `getattr(bundle, "routed")` | `REF<TS<i64>>` |
| `bundle["plain"]` | `TS<i64>` |

The graph's `TS<i64>` output shows `1` then `2`: an output declared as a
value, bound to a reference, follows it.

**Why.** WIR-5: a part known while describing is a structural projection,
with no node and no variable. WIR-13: a field declared as a reference
stays a reference.

## WIRE-PROJECTION-THROUGH-REF — WIR-5

*A part beneath a reference is not known until the graph runs.*

**Setup.** A routing node `if_(condition, value)` publishes a reference to
a bundle of references: which bundle it designates depends on the
condition. The graph selects the `true` field of that output. The condition
is true at cycle 0 and false at cycle 1; `value` is `1` then `2`.

| Question | Answer |
|---|---|
| The type of the selected part | `REF<TS<i64>>`, published by a node that wiring adds for the purpose |
| The graph's `TS<i64>` output at cycles 0 and 1 | `1`, then `—` |

At cycle 1 the condition turns false, the selected field's reference
becomes empty, and the follower unbinds. Unbinding is not a tick (TS-15,
TS-17), so nothing is published.

**Why.** WIR-5: beneath a reference the part is known only at run time, so
selecting it adds a node that publishes a reference to it.

## WIRE-SPECIFICITY — WIR-16, WIR-18

*The most specific matching candidate wins.*

**Setup.** An operator `pick` has three candidates, each publishing a label
that says which one ran:

| Candidate's parameter | Publishes |
|---|---|
| `TS<i64>` | `"int"` |
| `T` (any time-series) | `"generic"` |
| `TSL<T, n>` (a list of any time-series, any size) | `"list-generic"` |

| Call | Selected |
|---|---|
| `pick(TS<i64>)` | `"int"`: a concrete type beats a variable |
| `pick(TS<f64>)` | `"generic"`: only the variable matches |
| `pick(TSL<TS<i64>, 2>)` | `"list-generic"`: a variable inside a structure beats a bare one |
| `pick(REF<TS<i64>>)` | `"int"`: a reference adds no specificity and matches as its target |

## WIRE-FAILURES — WIR-4, WIR-16

*A call that cannot be resolved fails there, and fails the graph.*

| Situation | Result |
|---|---|
| An operator with two candidates that both take `TS<i64>`, called with `TS<i64>` | fails: ambiguous, and the error names both |
| An operator `only_int` with one candidate taking `TS<i64>`, called with `TS<str>` | fails: no candidate |

**The error's content.** A graph `outer` calls a graph `inner`, which makes
the failing call to `only_int` with a `TS<str>`. The error names: the
operator as its author declared it (`only_int`); the argument's type
(`TS<str>`); the candidate's parameter type (`TS<i64>`); and the path of
graph calls that led there (`outer`, then `inner`).

**A caught failure.** A graph `g` calls a graph `attempt`, which adds a
node and then makes the failing call; `g` catches the error and returns its
own input instead. Describing `g` still fails: what `attempt` already added
cannot easily be undone, so the session is failed (WIR-4).

## WIRE-REPEATED — WIR-7, WIR-17

*A variable that appears twice binds once.*

**Setup.** A node `same<T>(a: T, b: T)`.

| Call | Result |
|---|---|
| `same(REF<TS<i64>>, TS<i64>)` | wires: the first argument binds `T` to `TS<i64>` with its reference removed, and the second is the same type |
| `same(TS<i64>, TS<f64>)` | fails: the second argument does not match the bound `T` |

## WIRE-BUNDLE-IDENTITY — WIR-15, WIR-17

*Fields decide, unless both bundles are named.*

**Setup.** `Foo` and `Bar` are named bundle types with the same single
field `a: TS<i64>`; `{a}` denotes an unnamed bundle with that field. Nodes:
`same<T>(a: T, b: T)` as above; `takes_foo(x: Foo)`; `takes_unnamed(x: {a})`.

| Call | Result | Because |
|---|---|---|
| `same(Foo, {a})` | wires | one is unnamed, so fields decide |
| `same(Foo, Foo)` | wires | same named type |
| `same(Foo, Bar)` | fails | both named, different names |
| `takes_foo({a})` | wires | an unnamed bundle matches a named one by its fields |
| `takes_unnamed(Foo)` | wires | likewise |
| `takes_foo(Bar)` | fails | both named, different names |

**Field order.** `Pair` is a named bundle with fields `a` then `b`, both
`TS<i64>`; `Riap` a named bundle with the same fields declared `b` then `a`;
`{b, a}` the unnamed bundle with those fields. Sources publish `a = 1` and
`b = 2`; `takes_pair` and `takes_ba` each publish `a * 10 + b`, so a wrong
pairing would show.

| Call | Result |
|---|---|
| `takes_pair({b, a})` | wires and publishes `12`: fields pair by name, not position |
| `takes_ba(Pair)` | wires and publishes `12` |
| `same(Pair, {b, a})` | wires |
| `takes_pair(Riap)` | fails: both named, different names |

## WIRE-OPERATOR-CONTRACT — WIR-21 to WIR-24

*A candidate may extend and narrow its operator, never widen it.*

**Setup.** Four operators, each with the candidates shown. Every candidate
publishes a label so that the selection is visible.

| Operator, as declared | Candidates |
|---|---|
| `declares_generic(ts: T)` | one: `(ts: TS<i64>, scale: i64 = 2)`, with a parameter the operator does not declare and a default |
| `refinable(ts: T)` | one: `(ts: TS<i64>)`, narrower than the operator |
| `needs_extra(ts: T)` | two: a fallback `(ts: T)`, and `(ts: TS<i64>, scale: i64)` whose extra parameter has no default |
| `declares_int(ts: TS<i64>)` | a candidate `(ts: T)` is offered for registration: wider than the operator |

| Action | Result | Rule |
|---|---|---|
| `declares_generic(TS<i64>)` | publishes `extra 2`: the undeclared parameter takes its default | WIR-22 |
| `declares_generic(TS<i64>, scale = 5)` | publishes `extra 5`: the call supplied it | WIR-22 |
| `refinable(TS<i64>)` | publishes `refined`: the narrower candidate is selected | WIR-23 |
| registering `(ts: TS<i64>, scale: i64)` for `needs_extra` | accepted: a candidate may require what the operator does not declare | WIR-22 |
| `needs_extra(TS<i64>)` | publishes `fallback`: the candidate that requires `scale` does not match a call without it | WIR-22 |
| `needs_extra(TS<i64>, scale = 3)` | publishes `scaled 3`: both match, and `TS<i64>` is more specific than `T` | WIR-18, WIR-22 |
| registering the wider candidate for `declares_int` | rejected | WIR-23, WIR-24 |
| `declares_int(TS<f64>)` | fails: no candidate remains | WIR-16 |

The HGL modules under [runtime/validation/wiring](../runtime/validation/wiring/)
state the same expectations for the HGL front end: an implementation with a
`const scale` parameter its operator does not declare, with a default
([contract_superset.hgl](../runtime/validation/wiring/contract_superset.hgl))
and without ([contract_required_extra.hgl](../runtime/validation/wiring/contract_required_extra.hgl)),
must be accepted; an argument passed through the operator's call for such
a parameter ([contract_extra_argument.hgl](../runtime/validation/wiring/contract_extra_argument.hgl))
must be accepted; a wider implementation
([contract_widening.hgl](../runtime/validation/wiring/contract_widening.hgl))
must be rejected.

## WIRE-FRONT-END — WIR-14, WIR-7

*Every front end binds as the shared rules say.*

**Setup.** The HGL module [front_end.hgl](../runtime/validation/wiring/front_end.hgl)
declares a generic `pass<T>(value: T) -> T` and calls it with the argument
types below. The compiler resolves these calls itself, and must reach the
bindings the runtime would.

| Argument type | `T` must bind |
|---|---|
| `ref<f64>` | `f64` |
| `list<ref<f64>, 2>` | `list<f64, 2>` |
