# Base layer

Status: proposed, 2026-10-18.

This record defines the base layer: the language plus the mandatory core
module, `core`. It also defines the conformance pack, the spec's
executable cases, which are written against the base layer only.

**Agreed constraints:**
- Conformance tests do not depend on the standard library.
- The base layer is the core language, its testing framework, and the minimal
  functionality the standard library needs from below.
- The standard library, `std`, builds on the base layer and may add candidates
  for core operators while preserving their identities and contracts.
- `core` owns both operator declarations and required baseline implementations.
  Core implementation helpers live in `core.native`; additional helpers needed
  by `std` live in `std.native`.
- A conformance test needs to know only the behaviour an operator expression
  must have, not which candidate is selected. Overloads must stay stable for
  the expressions the tests write (§3, BASE-16).
- A conformance run loads no library: not the standard library, and not any
  other (BASE-12).

**Agreed conformance constraints:**
- Shared fixtures are shared. No copies are kept in step by a lint.
- The core's conformance tests form one module, `conformance`,
  assembled from parts, so its fixtures and tests share one flat namespace.
- The fixtures are that module's public declarations. A separate conformance
  module imports the ones it needs.
- The module exists only for conformance. No runtime ships or links it; only a
  conformance run supplies it.
- Rejection cases form a second module, `conformance.rejections`.

The remaining additions in this record are proposed. Existing rules retain
their current status.

## Layers

| Layer | Contains | Specified by | Realised by |
|---|---|---|---|
| Language | Grammar, types, phases, the test language | [Syntax and semantics](../developer-guide/syntax-and-semantics.md) and the design records | Each compiler |
| Core module `core` | Core operator declarations and baseline implementations; proposed prelude, injectables, language-form realisation, eval support and test host | [Core operator policy](operators.md#core-operators-and-the-standard-library) and this record | Each engine; a reference realisation in hgraph_spec `core/` is proposed |
| Standard library `std` | Its own operator declarations and additional candidates for core operators | [Library contracts](../../../library/README.md) and hgraph_std | Portable HGL plus selected native providers |
| Conformance pack | The spec's executable cases, as the module `conformance` | `conformance/README.md` | hgraph_spec `conformance/` |

The **base layer** is the language plus the core. Library HGL is written
against the base layer only. Library *native* parts may use runtime
facilities directly: that is how the nested-graph APIs and view helpers exist.

## The core module

`core` is the mandatory module of the
[module boundary](modules.md#boundary). It owns both the core operator
declarations and their required baseline implementations. Core implementation
helpers live in `core.native`; additional standard-library helpers live in
`std.native`. A selected core helper provider is part of core's implementation
dependency closure and introduces no dependency on `std`.

Eval support and language-form realisation below remain proposed. Module parts
([ADR 0006](decisions/0006-multi-file-module-parts.md)) and native implementation
parts ([ADR 0014](decisions/0014-native-implementation-interfaces.md)) let their
portable and native declarations share one module identity.

| Surface | Reached by | Contents |
|---|---|---|
| Prelude | Unqualified lookup, step 5 only | §1 |
| Public interface | `use core::{...}` or a module alias | Core operator contracts, including essential named operations; proposed `TimedValue`, `replay` and `record` (§5) |
| Token binding | Expression syntax | The 18 identities of the [fixed mapping](operators.md#fixed-symbol-to-name-mapping) |

Source cannot name how a language form is realised, nor which replay and record
realisation eval wires. Neither is a surface.

### 1. Prelude

| Group | Names | Contract |
|---|---|---|
| Endpoint metadata | `valid`, `modified`, `all_valid`, `last_modified` | Language model; temporal metadata |
| Delta | `delta_value` | [Delta value metadata](delta-value-metadata.md) |
| Collection views | `key_set`, `keys`, `values`, `elements`, `items`, `added`, `removed` | Traversal; [iteration](iteration.md) |
| Output mutation | `insert`, `update`, `upsert`, `remove`, `discard`, `invalidate`, `clear`, `push`, `pop` | Runtime output; the first argument is `out` |
| Activation (reserved) | `scheduled`, `passivate`, `activate` | [ADR 0010](decisions/0010-lifecycle-capabilities.md) |
| Capability calls | `schedule`, `schedule_at`, `is_scheduled`, `next_scheduled_time`, `info`, `get`, `set` | [Capability function syntax](capability-function-syntax.md) |
| Value operations | `len`, `at`, `get`, `contains`, `push` | Ordinary lists; BYTE-4; LOOKUP-1–4 |
| Outcomes | `ok`, `err`, `error`, `is_ok`, `is_err`, `capture` | [Result errors](result-errors.md); [error capture](error-capture.md) |
| Keyed results | `map` | §4, MAP-1–MAP-7 |

The value-level `len` covers `str`, `bytes`, and ordinary lists, sets and maps.

The following helpers remain outside the proposed prelude and are reached
through `core.native`, following the [native helper contract](../user-guide/modules-and-tools.md#core-utility-functions):
- `bound` and `active`;
- `is_empty`;
- `len` and `is_empty` on temporal views;
- `front` and `back`;
- `schemas`;
- the window metadata.

Additional helpers needed only by `std` belong in `std.native`.

Name resolution, in order:
1. The reserved intrinsics `scheduled`, `passivate` and `activate` are not
   identifiers and cannot be shadowed
   ([ADR 0010](decisions/0010-lifecycle-capabilities.md)).
2. A direct capability receiver selects the capability meaning, even when a
   library name with the same spelling is imported (existing,
   [capability function syntax](capability-function-syntax.md)). The
   receivers are `alarm`, `scheduler`, `logger`, `global_state` and `out`.
3. Lookup steps 1–4 shadow every other prelude name (existing).
4. Any remaining choice is made by context:
   - inside `requires` or in a body;
   - the containing phase, for the metadata names;
   - the operand kind (enum type, ordinary value).

Prelude names are not reachable through a module alias
([syntax and semantics](../developer-guide/syntax-and-semantics.md), Scopes
and name lookup).

### 2. Injectables

The injectables are `out`, `logger`, `clock`, `scheduler`, `alarm` and
`global_state`. Each one is available in every phase where the language
admits it.

Some things have no HGL spelling yet: engine control, `clock.lag`, logger
levels, and scheduler tags. None of them is base until it has one.

### 3. System operators

The sixteen declarations below cover the syntax-bound arithmetic, comparison
and Boolean operators. `and_`, `or_` and `not_` constrain their operands to
`bool` (proposed; see **Logical operators**).

This is not the complete core inventory. The [core operator policy](operators.md#core-operators-and-the-standard-library)
also includes essential named operations such as `core.abs`. Its required
roles and domains still need a detailed contract (open point 18); this proposal
does not assign it to `std`.

```hgl
operator add_<L, R, O>(lhs: L, rhs: R) -> O
    properties<str, str, str> { associative, identity = "" }
    properties<i64, i64, i64> { commutative, identity = 0 }

operator sub_<L, R, O>(lhs: L, rhs: R) -> O

operator mul_<L, R, O>(lhs: L, rhs: R) -> O
    properties<i64, i64, i64> { commutative, identity = 1 }

operator div_<L, R, O>(lhs: L, rhs: R) -> O

operator floordiv_<L, R, O>(lhs: L, rhs: R) -> O

operator mod_<L, R, O>(lhs: L, rhs: R) -> O

operator neg_<T, O>(ts: T) -> O

operator eq_<L, R>(lhs: L, rhs: R) -> bool

operator ne_<L, R>(lhs: L, rhs: R) -> bool

operator lt_<L, R>(lhs: L, rhs: R) -> bool

operator le_<L, R>(lhs: L, rhs: R) -> bool

operator gt_<L, R>(lhs: L, rhs: R) -> bool

operator ge_<L, R>(lhs: L, rhs: R) -> bool

operator and_<L, R>(lhs: L, rhs: R) -> bool
requires L == bool && R == bool
    properties<bool, bool> { associative, commutative, identity = true }

operator or_<L, R>(lhs: L, rhs: R) -> bool
requires L == bool && R == bool
    properties<bool, bool> { associative, commutative, identity = false }

operator not_<T>(ts: T) -> bool
requires T == bool
```

`getitem_` and `getattr_` are core identities with no source declaration.

Each implementation binds these identities to its own native operators: the
target binding of [modules.md](modules.md), Nominal operator binding.
Declaring them creates no new runtime operator. The candidates an
implementation designates for the domains below are the core's
**realisation**. The core descriptor names them, so `hgl check` and a native
build see the same realisation ([modules.md](modules.md), Candidate universe
and provider discovery).

The core fixes the behaviour of these identities over the domains below, not
their candidate set. Implementations and libraries may add
candidates: more domains, native nodes, and HGL `impl fn` declarations of an
imported core identity ([modules.md](modules.md), Nominal operator binding).
Every candidate comes from HGL's closed universe: the target's modules, its
locked dependency closure, the core and the selected native packages. A
candidate outside the target's declared packages, such as an overload a host
registers at run time, is outside it ([modules.md](modules.md), Candidate
universe and provider discovery). A conformance run loads no library
(BASE-12), so the candidates a case meets are the realisation and the pack's
own declarations.

**Core-domain calls.** A core-domain call is a call to a core identity,
by token or by name, that:
- supplies the operands of a row below and no other argument;
- has operand types that the row covers.

Membership is decided on the operand types at the call, not on candidate
signatures, so a generic candidate stays live outside the domains.
- At value level, an operand is an ordinary value of the row's type.
- In composition, an operand is a time series whose payload is one atomic
  value of the row's type, a reference to one (WIR-6, WIR-8), or a constant
  lifted to one. Structural temporal operands are outside the core domains:
  non-atomic lists, structs, maps and sets, and windows.
- Every use of a mapped token is a call of its core identity, whatever
  `requires` clause is active. [Operators](operators.md) already says a local
  function named `add_` does not change `a + b`.
- A call that passes an extra argument, such as an explicit tolerance
  (WIR-22), is not a core-domain call.

**Roles.**

| Role | Requirement |
|---|---|
| Value level | Native-required, over the domains below. The spec has no form for value-level candidates yet ([operators](operators.md), Execution role and native implementations; [ADR 0008](decisions/0008-temporal-contracts-and-target-mappings.md), Operators and native implementations), so node evaluation and folding use the core's own operations |
| Composition | One node per call, equivalent to `when valid(lhs, rhs) { return f(lhs, rhs) }`, or `when valid(ts)` for a unary identity. The exception is a row that says otherwise |
| Constant folding | Value-level evaluation at compile time, except where a row forbids it |

The composition node keeps the selections of ADR 0008's lifted node
([ADR 0008](decisions/0008-temporal-contracts-and-target-mappings.md), Role
selection and default lifting; [functions](../user-guide/functions.md)) and
writes its validity requirement out:
- A notification from any operand activates it, including an invalidation.
- It runs only when every operand is valid.
- When an operand is invalid, its output is invalidated without running the
  body. An explicit validity requirement requests this
  ([functions](../user-guide/functions.md)), and it is what OP-1 requires.
- It publishes on every evaluation it runs, equal to the last result or not
  (proposed). OP-4 lets a deriving operator elide an unchanged recomputation
  ([operator contracts](../../../library/operator_contracts.md)). That
  permission does not apply here, because two realisations could then publish
  different ticks.

This node is the reference behaviour. An implementation may realise
composition another way, such as a native node, if the call's behaviour (see
**Stability**) is the same.

OP-1 (invalidity) and OP-3 (no input mutation) move from
[operator contracts](../../../library/operator_contracts.md) and become core
rules. String conversion follows the [HGL conversion contract](../user-guide/types-and-expressions.md#string-conversion);
OP-9 leaves map and set member order unspecified.

Where a row admits several roles, they give the same result for the same
operands. [Syntax and semantics](../developer-guide/syntax-and-semantics.md)
(Arithmetic and comparison) and [operators](operators.md) already require
this of the temporal and floor rows. The core extends it to every row
(proposed).

**Domains.**

| Operators | Domain | Roles |
|---|---|---|
| `add_`, `sub_`, `mul_` | `i64` gives `i64`; `f64` and mixed `i64`/`f64` give `f64` | All |
| `div_` | Numeric gives `f64` | All |
| `floordiv_`, `mod_` | `i64` gives `i64`; `f64` and mixed give `f64`; floor rules ([operators](operators.md), Signatures, lifting, and implementation) | All |
| `neg_` | `i64`, `f64`, `duration` | All |
| `add_` | `str` concatenation | All |
| `add_`, `sub_`, `mul_`, `div_` | The rows of the temporal arithmetic table ([syntax and semantics](../developer-guide/syntax-and-semantics.md), Arithmetic and comparison) | All. Zoned rows never fold |
| `eq_`, `ne_` | Every same-type pair with the equality capability (VAL-9) | All. `any` follows ANY-3 |
| `eq_`, `ne_` | `ref<T>` | Node evaluation only (REF-EQ-1) |
| `lt_`, `le_`, `gt_`, `ge_` | Every same-type pair with the order capability, including `bool` (BOOL-1), `any` (ANY-3) and native atomic types that declare order | All |
| `and_`, `or_`, `not_` | `bool` | All |
| Every arithmetic and comparison identity | A temporal operand in a combination the temporal table does not list | Rejected with a `type` diagnostic |
| `lt_`, `le_`, `gt_`, `ge_` | A same-type pair whose type lacks order, such as sets and maps (VAL-11) | Rejected. Phase and code are open (open point 14) |
| `getitem_`, `getattr_` | Projection that is not structural, of an atomic tuple, list, struct or map payload | Composition. At value level the type's own rules apply, such as LOOKUP-1–4 for maps |
| `getitem_`, `getattr_` | A map key giving `ref<V>`; projection beneath a REF | Composition. These follow WIR-5, MAP-CONN-1–4 and TS-16, not the lifted node: the REF ticks only on a changed designation |

**Equality.** Equality is exact. A core-domain `==` is true only for equal
values, with NaN as [NaN comparisons](nan-comparisons.md) says, and `!=` is
its negation (proposed; REF-EQ-1 already says so for references). No
core-domain call applies a tolerance, whichever candidate realises it. A
tolerant comparison is a named operator with its own parameter, such as the
standard library's `eq_epsilon`.

Comparisons between `i64` and `f64` are not a row (open point 12). Pack cases
do not use them.

**Result type** (proposed). The row fixes the result type. For a
core-domain call, an expected type does not request another one: it binds
no result parameter, unlike an ordinary call
([type extensions](type-extensions.md); WIR-12). Ordinary conversion, such as
`i64`-to-`f64` widening, then applies at the destination
([contextual local bindings](contextual-local-bindings.md)).

**Failures.** A failure keeps the form its own rule gives it:
- The `any` rows follow ANY-3. An executed operation fails with
  `value.capability`, a required constant with `value.constant_capability`,
  and optional folding keeps the execution failure
  ([any values](any-values.md)).
- Temporal overflow in a constant expression is a compile-time diagnostic
  ([syntax and semantics](../developer-guide/syntax-and-semantics.md),
  Arithmetic and comparison).
- A zero divisor and `i64` overflow are errors whose code and phase are open
  ([operators](operators.md); [language model](language-model.md); open
  point 14). Pack cases avoid them.

**Stability.** *(Agreed.)* A conformance case asserts the
behaviour an operator expression must have, never which candidate is
selected. Overloads must not change that behaviour for the expressions the
tests write.

The core makes this precise (proposed). In a conformance run no candidate
changes the behaviour of a core-domain call: the realisation gives it,
whatever candidates it holds beyond the table, and the pack's own candidates
stay off core rows (Pack check). That behaviour is:
- its output trace: values, deltas, ticks, modification times and validity
  ([runtime conformance](../../../runtime/conformance.md));
- its result type;
- its folded value, where it folds;
- whether it is accepted. No candidate makes it ambiguous, rejected or newly
  accepted, or changes whether a `requires` over it holds and what that
  requirement binds ([language model](language-model.md));
- a failure's catalogued code, and whether the failure is a source rejection
  or an execution failure.

A candidate that does not have the operator's shape is rejected when it is
registered (WIR-24). That fails its own module, not the call.

Which candidate realises a call, and the nodes it adds, are not part of the
call's behaviour:
- whether an implementation is a graph or a node is invisible at the call
  site ([types and expressions](../user-guide/types-and-expressions.md));
- a target binding may choose its symbols "without changing those
  obligations"
  ([ADR 0008](decisions/0008-temporal-contracts-and-target-mappings.md),
  Target boundary).

So the trace compared for a core-domain call is its output, not what runs
inside it. For a core identity, the "operator selection" in ADR 0008's
acceptance list means selecting this behaviour.

**The realisation.** It has exactly one most specific candidate for every
core-domain call, and the core descriptor names it. A tie inside it is the
implementation's nonconformance, not a permitted rejection.

**Library candidates.** A conformance run loads no library, so no library
candidate reaches a case. In a program that loads `std`, its candidates retain
the operator's core identity and applicable contract. Ordinary matching and
specificity apply: library membership adds no override priority, and equally
ranked applicable candidates are ambiguous (WIR-16). Further cross-module
coherence rules remain open; this record adds no replacement mechanism
(open point 15).

**Not core calls.** These spellings are language forms with their own rules.
They are never resolved among candidates, and no candidate changes them:
- the harness comparison `eval(...) == [...]`
  ([syntax and semantics](../developer-guide/syntax-and-semantics.md);
  [eval operator composition](eval-operator-composition.md));
- a presence test against `null`
  ([nullable replay indexing](nullable-replay-indexing.md));
- type equality and the connectives `&&`, `||` and `!` inside a `requires`
  clause ([syntax and semantics](../developer-guide/syntax-and-semantics.md));
- a structural projection (WIR-5).

**Outside the core domains.** An implementation or library may give an
operator expression outside the table a meaning, or none. The pack asserts
neither acceptance nor rejection of such an expression, with one exception.
When a case's own part declares a type and a concrete candidate over it, WIR-18
selects that candidate whatever else the universe holds, so the case may assert
its result. The rejected rows above are core-domain calls, not extensions: no
candidate makes them accepted.

**Logical operators** (proposed). `and_`, `or_` and `not_` admit only `bool`,
in every phase and by token or by name. A candidate on another type is
rejected when it is registered (WIR-23, WIR-24). Three things follow:
- a token and a named call of one identity agree ([operators](operators.md));
- node evaluation, which short-circuits on `bool` ([operators](operators.md)),
  and composition agree;
- a `requires` naming them binds only `bool`.

Truthiness on other types belongs to a separately named operator.

### 4. Language-form realisation

The language rule for each form is its whole observable contract. A form that
owns children obeys [runtime/graph.md](../../../runtime/graph.md), "A graph
owned by a node" and "What a nested node may do". Which runtime operator
realises a form is not part of the base layer.

Each implementation must supply these facilities natively:

| Facility | Language forms |
|---|---|
| Conditional child graph | Temporal `if`, temporal `switch`, an outputless conditional in composition |
| Keyed child graph | Composition `for` over a dynamic map or list (outputless bodies); `map(...)` over one map, with a keyed result |
| Capture boundary | `capture(...)` (CAPTURE-1–5) |
| Live key-set projection | Composition `key_set` |

The other lowerings are ordinary nodes that can be written in HGL:
- a never-ticking typed branch;
- composition metadata;
- constant lifting;
- conversions.

#### Keyed results: `map(...)`

Proposed. These rules may move to their own record, `higher-order-map.md`.

The spec calls `map(...)` in three examples
([functions](../user-guide/functions.md),
[language tour](../user-guide/language-tour.md),
[structural-types.hgl](../../examples/structural-types.hgl)) but gives it no
contract ([iteration](iteration.md), [control flow](control-flow.md)). The
type grammar has no callable type
([syntax and semantics](../developer-guide/syntax-and-semantics.md)), so no
module can declare it. Like `capture` (CAPTURE-1), it is a prelude form whose
result type depends on a callable argument. It is the keyed child graph with a
result.

- **MAP-1 Form.** `map(c, fn(p) => body)`, in graph composition. `c` is a
  non-atomic temporal `map<K, V>`. The second argument is an anonymous
  function, and `map` is the context that fills its types
  ([syntax and semantics](../developer-guide/syntax-and-semantics.md),
  Anonymous functions): `p` has type V, and a written annotation must equal
  V. The result type R is the written `-> R`, or else the type of the body.
  The result is `map<K, R>`. An outputless body is refused: that is `for`.
- **MAP-2 Membership.** There is one child per key of `c`. A key is a member
  of the result from its child's creation until that child stops, whatever
  the validity of the child's output (TS-19, MAP-CONN-2). Creation adds the
  key in that cycle. A child output that becomes valid or invalid neither adds
  nor removes the key.
- **MAP-3 Identity and lifetime.** These follow dynamic `for`
  ([iteration](iteration.md)). Updating a member keeps its child. When its key
  leaves `c`, the child stops (GRF-18, GRF-23). A later insertion creates a
  fresh child with fresh state.
- **MAP-4 Result deltas.** A key's member is its child's output (GRF-10). A
  child's publication is that key's upsert in the same cycle (GRF-15,
  GRF-21). When a child stops, its key is removed in that cycle (TS-19,
  GRF-25), and none of its pending work runs (GRF-20; the `cancel_timer`
  case of [nested graph cases](../../../runtime/cases_nested.md)). A
  sibling's tick republishes no member.
- **MAP-5 Captures.** The function may name temporal bindings of the
  enclosing composition. Each one is bound whole to every child and adds no
  keys. A rebind reaches existing children without recreating them (GRF-10).
  This is the `for` rule ([iteration](iteration.md);
  [dynamic-collection-iteration.hgl](../../examples/dynamic-collection-iteration.hgl)),
  and it is how KEYED-ROUTE-TIMER passes its price dictionary to each child
  ([nested graph cases](../../../runtime/cases_nested.md)). A `const` capture
  is not promoted to a time series. Scalar capture identity and binding stay
  open, as they are for `for`.
- **MAP-6 Independence.** Children share nothing but their owner (GRF-24). A
  child's next scheduled time schedules its owner, and each enclosing owner in
  turn (GRF-22).
- **MAP-7 Name.** `map` is a prelude name. A declaration named `map` in a
  module shadows it there (§1, step 3). No module re-exports the core's
  `map`.

For the structured test dump (§6), a call inside the anonymous function is a
call made by the function that holds the `map(...)`.

These are outside the contract (open point 8): more than one collection, a key
parameter, list collections, a named-function argument, and a `for` with a
result.

### 5. Eval support

The core declares
`export struct TimedValue<T> { time: datetime; value: delta<T> }` and the
replay and record contracts of
[ordinary replay and record](../../../library/ordinary_replay_record.md),
together with OP-11. Those contracts move from `library/` to core status.

- **Binding.** Eval binds the core's own replay and record realisation.
  - Another module may add candidates to them. Eval does not choose among
    candidates, so an added candidate is never what eval wires. A direct
    call resolves normally
    ([eval operator composition](eval-operator-composition.md)).
  - A test module's imports and local declarations do not change what eval
    wires.
  - Library code and users may still call them directly.
- **Timed sequences.** A `duration` key is the run start plus that duration. A
  `datetime` key is the entry time itself.
- **Borrowing.** `var recording = get(global_state, key)` is a writable borrow,
  so `push` lands in the store ([value mutability](value-mutability.md)). This
  is required behaviour, not an optimisation.
- **Reference realisation** (proposed). hgraph_spec `core/` will hold
  `TimedValue`, `replay` and `record` in HGL. The portable source bodies will be
  relocated without semantic changes. An implementation may substitute a native realisation
  that passes the core cases.
- **Closure.** A reference body uses only:
  - the language;
  - prelude names;
  - injectables;
  - core declarations and their selected `core.native` helpers.

  A body that needs anything else first amends this record.

### 6. Test host

| Facility | Contract |
|---|---|
| Eval driver | EVAL-1–6; [recorder keys](eval-recorder-keys.md); dense materialisation; timed sequences |
| `assert` | Takes a wiring-time `bool`. A failure reports the first differing cycle. Harness equality is not a core call: it is exact, compares presence first and follows the harness rules of each shape ([syntax and semantics](../developer-guide/syntax-and-semantics.md)). No candidate changes it |
| `assert raises` | [Execution-error assertions](execution-error-assertions.md); the [error catalogue](error-catalogue.md) |
| `# expect-error` | [Source-rejection tests](compile-rejection-fixtures.md), including owner isolation |
| Test scopes and attributes | One nestable `test [attributes] { ... }` scope, which holds helpers, nested scopes and statements. `assert [attributes] ...` has the same form. The attributes are `case: ID`, `rules: ID, ...` and `scenario: "text"`. They are checked and inherited downwards: `case` is set once per branch, `rules` accumulate, and `scenario` extends the path. A test name is optional. Every scope, attributed or not and in any part, joins the module-wide test scope: a scope groups source and is not a namespace ([testing and running](../user-guide/testing-and-running.md)). A case › scenario path is unique in the module. Inside an unnamed scope, the owner of an `# expect-error` is the innermost scope that sets `case` or adds a `scenario` |
| Report | Per test: the outcome, the failure kind and the code. Identity is the case › scenario path |
| Structured test dump | Versioned. Emitted even when rejection owners are present. One dump covers every part of an assembled module. It carries the scope tree, the attributes, the steps with eval arguments bound to parameter names, and the cells. Each declaration, scope and rejection record also carries its part and file, its canonical identity and the declarations it resolves to. A part that fails checking still gets a record, with its source and its diagnostics, and the other parts' records stay complete. It is the documenter's only input |

## Conformance pack

- **Location.** The pack lives in hgraph_spec `conformance/`, versioned with
  the core. A conformance claim names one hgraph_spec revision.
- **One module.** *(Agreed.)* The pack's tests form one
  module, `conformance`, assembled from parts
  ([ADR 0006](decisions/0006-multi-file-module-parts.md); [modules.md](modules.md),
  Source module parts). Its parts share one declaration scope and one test
  scope, in any order
  ([testing and running](../user-guide/testing-and-running.md);
  [syntax and semantics](../developer-guide/syntax-and-semantics.md)), so a
  part uses a fixture without importing it. The module is not
  `core`, so its tests see only the core's public interface and the
  prelude. **Separate modules** lists the cases kept out of it.
  - **Test-only.** *(Agreed.)* The module exists only for conformance. No
    implementation ships, registers or links `conformance` or a module
    named under it. Only a conformance run supplies them, so an import of one
    anywhere else is unresolved (`module.unresolved`).
  - `conformance/conformance.hgl` is the module's one unnamed file
    ([ADR 0006](decisions/0006-multi-file-module-parts.md)) and the anchor of
    every run. It holds the module's documentation and the pack's only `use`
    line, and declares nothing. Every run includes it, so whatever it declared
    would be required of every run.
  - The fixture parts hold the fixtures. Every other part is one case file,
    named after its file stem with `-` written `_`. Its documentation
    comment titles its page.
  - `conformance/pack.json` owns the part list (proposed), as a build target
    would ([ADR 0006](decisions/0006-multi-file-module-parts.md)). It follows
    the [negative-testing manifest](../../../compiler/negative_testing/cases.json),
    whose schema 2 lists `parts`
    ([negative-testing conformance](../../../compiler/negative_testing/README.md)).
    For each case part it lists the fixture parts the part uses. It also lists
    the separate modules and their partners. `conformance/README.md` describes
    it and does not copy it.
  - Each case has one encoding, in the spec's spelling. A lag variant, such
    as a pinned or dense copy of a case, is not a pack file. An
    implementation that cannot check a part yet lists it in its profile, and
    a profile that lists a fixture part lists every part that uses it.
- **Fixtures.** *(Agreed.)* A fixture is a function that two or more case parts, or a
  separate module, use. It is declared once, as an `export fn` at module level
  of a fixture part, and never copied. A helper that one case part uses stays
  in that part's test context
  ([testing and running](../user-guide/testing-and-running.md)).
  - A fixture is a production declaration, because test helpers cannot be
    exported or imported
    ([syntax and semantics](../developer-guide/syntax-and-semantics.md)). It
    cannot name a test-local type. The module is test-only, so its public
    interface reaches only conformance modules.
  - Part `fixtures` holds the general fixtures. A fixture that uses a type of
    one of these families, in its declaration or through the fixtures it
    calls, lives in that family's part instead: `fixtures_native` (native
    types), `fixtures_any` (`any`), `fixtures_window` (windows),
    `fixtures_reference` (`ref<T>`) and `fixtures_keyed` (sets and maps).
    The first family in that list decides. A run then needs only the types
    its case needs: an implementation without `any` excludes `fixtures_any`
    and its users and still runs the rest.
  - Fixtures are plain functions, generic where the spec's text is.
    `pass_through<T>` is the [delta value metadata](delta-value-metadata.md)
    fixture verbatim, declared `export fn` as
    [reference-routing.hgl](../../examples/reference-routing.hgl) exports a
    generic function, and `consume(tick: i64)` is the
    [testing and running](../user-guide/testing-and-running.md) sink.
  - Eval takes a module `fn`
    ([testing and running](../user-guide/testing-and-running.md)), so a case
    wraps a generic fixture in a typed `fn`. A wrapper that two case parts
    use is a fixture too. It is named `pass_through_<type>`, with the type
    written inside out, as
    [eval operator composition](eval-operator-composition.md) names
    `pass_through_i64`: `list<i64, 2>` gives `pass_through_i64_list2`. Two
    authors of one wrapper then choose one name, and the full assembly
    reports the duplicate.
  - A second module-level declaration of a fixture's name is a duplicate,
    which the full assembly rejects
    ([ADR 0006](decisions/0006-multi-file-module-parts.md);
    [syntax and semantics](../developer-guide/syntax-and-semantics.md)). A
    test helper with a fixture's name shadows it legally, and the pack check
    rejects that.
  - A `const fn` fixture stays unexported, because `export const fn` has no
    accepted spelling ([functions](../user-guide/functions.md); open point
    17). The pack's parts still share it.
  - Types are not fixtures. A test-local type's identity is its module-wide
    name ([testing and running](../user-guide/testing-and-running.md)), so
    two equal `struct` declarations are two types, not a copy.
  - Case data is a literal where a parameter, field or element gives it its
    type ([ordinary list values](ordinary-list-values.md)). A constant that
    needs a declared type, such as an empty list boxed by `any(...)`, stays
    in its part until a second part needs it.
- **Native fixtures.** Part `fixtures_native` holds the native atomic fixtures
  of [native atomic value cases](../../../compiler/cases_native_atomic_values.md):
  three exported `native type` declarations, `Token`, `TokenOther` and
  `TextOnly`, and six `native const fn`s. They sit at module level, because a
  test context admits no native declaration
  ([syntax and semantics](../developer-guide/syntax-and-semantics.md)). The
  part also exports the fixture `observe_text`. The exports let a separate
  module import the types (NVAL-1).
  - A native library build selects one provider
    ([ADR 0014](decisions/0014-native-implementation-interfaces.md);
    [native implementation parts](native-implementation-parts.md)). Every
    native fixture therefore joins this part and its one provider.
  - Each implementation supplies the provider, test-only, in its own
    repository ([native implementation parts](native-implementation-parts.md)).
    `conformance/README.md` states each type's capabilities once, because
    every target satisfies the same contract (NVAL-2).
  - A run includes this part and the provider only when one of its parts
    names a native fixture. The full assembly always includes them.
- **Separate modules.** Three kinds of case are not parts of the pack module.
  Each is listed in `pack.json` with its partners and runs alone. A name under
  `conformance.` groups listings only, and gives no access to the pack
  module's private declarations
  ([ADR 0006](decisions/0006-multi-file-module-parts.md)).
  - Rejection cases (`# expect-error`) form one module,
    `conformance.rejections`, assembled from parts in the same way
    *(Agreed.)*. In the pack module, a boundary that an
    implementation's syntax recovery cannot isolate, or a malformed
    annotation, would fail admission for every case
    ([source-rejection tests](compile-rejection-fixtures.md)). Each rejection
    case is also checked against the whole surviving module, and an error it
    causes in any other declaration fails it.
  - A case whose subject is an import, an alias, an export, a descriptor or
    the run rule, `module.unresolved` among them, is its own module. The NVAL
    import, export and alias rows import the types of `fixtures_native`.
    Their partner is the pack module built from `conformance.hgl` and
    `fixtures_native`.
  - A shadowing case is its own module, because a declaration named like a
    prelude name shadows it in every part (§1, step 3; MAP-7).

  A separate module imports the fixtures it uses from `conformance`. Its
  partner is the pack module built from `conformance.hgl` and the fixture parts
  that declare them. The harness controls in `compiler/negative_testing/` fail
  by design and stay there
  ([negative-testing conformance](../../../compiler/negative_testing/README.md)).
- **Dependency.** A pack file imports only `core`, and a separate
  module also `conformance`. In the pack module only `conformance.hgl`
  has a `use` line. Imports join the whole module's scope
  ([ADR 0006](decisions/0006-multi-file-module-parts.md)), so a `use` in one
  part would serve the others silently. There are two exemptions:
  - a `use` owned by an `# expect-error`
    ([invalid-import.hgl](../../examples/reject/invalid-import.hgl));
  - a separate module's import of another partner that `pack.json` lists.
- **Runs.** *(Agreed: no library dependency.)* A pack run supplies only `core`, the files `pack.json`
  lists for that run and the pack's own native provider. It supplies no
  library source, descriptor or provider, so a green run proves that the pack
  does not need the library. Any `core.native` provider selected by core is
  included as part of core's declared implementation dependencies. No other
  run supplies `conformance`.
  - **Full assembly.** Required. A conformance claim runs `conformance.hgl`
    with every part, in one process per implementation, and names the parts
    its profile excludes. Only the whole set shows duplicate names
    ([ADR 0006](decisions/0006-multi-file-module-parts.md)).
  - **One case.** `conformance.hgl`, one case part and the fixture parts it
    uses, directly or through other fixtures. A part can be checked alone
    ([ADR 0006](decisions/0006-multi-file-module-parts.md)), so this is a
    conforming build, and it serves triage.
  - **Separate modules.** Each runs alone, with its partners.
- **Imports.** An import resolves only against the core and the modules the
  run supplies.
  - Resolving it against anything else, such as a machine-wide registry, does
    not conform ([modules.md](modules.md), Candidate universe and provider
    discovery).
  - A new catalogue code, `module.unresolved`, and a pack case test this rule.
    The case is a separate module.
- **Pack check** (proposed). The full assembly rejects a duplicate
  declaration, so the pack needs no copy lint. A small check rejects what the
  language allows but the pack does not. Until implementations emit the test
  dump (§6), it reads source. Names are unique across the module, so a name
  read outside a local binding, a field or a member access resolves to one
  declaration. It rejects:
  - an import outside the Dependency rule, or a `use` line in a part;
  - a case part that names another case part's declaration;
  - an `export` outside a fixture part;
  - an `impl fn` of a core identity with no operand of a type its own part
    declares. The parts share one scope, so such a candidate would reach every
    part's core-domain calls;
  - a declaration, other than an `impl fn`, that takes a prelude name (§1), a
    core public name or a fixture's name. The language lets it shadow
    ([syntax and semantics](../developer-guide/syntax-and-semantics.md),
    Scopes and name lookup), and the pack does not;
  - two functions that are equal up to renaming the declaration, its
    parameters and its generic parameters, with literals compared as
    written. This finds a renamed copy anywhere in the module, so it needs no
    list of exemptions. It cannot see a typed copy of a generic fixture,
    which is left to review;
  - a `case` ID in more than one part;
  - a profile that excludes a fixture part but not every part that uses it;
  - a name declared twice, until every implementation reports it.
- **Pages.** The documenter reads the test dump of the full assembly (§6).
  - Each part has one page, titled by its documentation comment. Its sections
    are its cases, in order of first appearance. Tests with no `case` go
    under "Other tests".
  - A page shows every fixture its tests reach, directly or through other
    fixtures: its signature, its summary and its source, collapsed. These
    are generated from the dump, so the source stays in one place and the
    page is complete.
  - The Fixtures page indexes the fixture parts, with links to the cases that
    use each fixture. Anchors are canonical identities, such as
    `conformance.pass_through` (proposed), so they survive moving a
    declaration between parts
    ([ADR 0006](decisions/0006-multi-file-module-parts.md)).
  - The documenter renders a page for every part `pack.json` lists. A part
    that the documenting implementation cannot check is shown from its source
    and marked as not checked.
- **Operator cases.** A case on a core identity asserts behaviour only
  (BASE-16, BASE-17).
  - It never asserts a candidate label, which implementation ran, the node
    identity in an error's context (ERROR-2) or an error's message.
  - It asserts nothing outside the core domains, except a result over a type
    its own part declares with a concrete candidate (BASE-18). The type and its
    `impl fn` are production declarations of the pack module, because a test
    context admits no operator implementation
    ([syntax and semantics](../developer-guide/syntax-and-semantics.md)), so
    their names are unique across the pack.
  - Where an identity has both a token and a public named declaration, it
    makes each assertion through both: `a + b` and `add_(a, b)`. Projection
    identities without source declarations are tested through their language
    forms; their public named interface remains open (open point 3).
  - WIRE-SPECIFICITY and WIRE-FAILURES observe selection, but only on an
    operator the case declares itself, so they stay valid.
- **Discriminating cases.** Operands that tell realisations apart are required:
  - one comparison in composition, in a node body and folded: `0.0 == 1e-11`
    is false in all three, and `x == x` is true for `x = 1e308 * 2.0`;
  - in composition over `a: [0.0]` and `b: [1e-11]`, `a == b` gives `[false]`
    and `a != b` gives `[true]`;
  - an equal result still publishes: `a * 0` over `a: [1, 2, 3]` gives
    `[0, 0, 0]`, and `a == b` over `a: [1, 2]`, `b: [5, _]` gives
    `[false, false]`;
  - OP-1: when a helper invalidates an operand, `a + b` is invalid in that
    cycle, as a `when valid()` probe that returns `valid(x)` observes.
- **Host-level cases.** A case that needs an observation or facility with no
  HGL spelling is host-level and sits outside the HGL pack. Today these are:
  - push sources and sender threads;
  - the controlled real-time timer;
  - physical-layout fixtures;
  - hash observation;
  - feedback;
  - `out` in start or stop;
  - printed output;
  - log capture;
  - lifecycle traces;
  - run-owner seeding and reading of `global_state`;
  - run configuration.
- **Blocked.** Refusals with no code cannot be written yet: the
  [`contextual_bindings`](../../../compiler/contextual_bindings) cases, the
  [`tuple_construction`](../../../compiler/tuple_construction) rejects,
  NOD-26, a past-time alarm request, operator ambiguity and no-match, the
  rejected rows of §3, a non-`bool` operand of `and_`, `or_` or `not_`, and
  arithmetic failures (open point 14).
- **Nested cases.** KEYED-ROUTE-TIMER and DEADLINES-AND-REMOVAL
  ([nested graph cases](../../../runtime/cases_nested.md)) are pack cases
  through `map(...)` (§4). They keep their `/ticks` assertions
  ([runtime/validation/assertions.json](../../../runtime/validation/assertions.json)),
  with three changes:
  - NOD-13 is not cited. HGL has no scheduler tags
    ([injectables](../../../runtime/injectables.md)). A deadline held in
    `state` gives the same ticks: a stale wake publishes nothing, instead of
    being replaced.
  - The `/events` assertions (GRF-18, NOD-11) are lifecycle traces, so they
    are host-level. A fresh child still shows in the ticks: KEYED-ROUTE-TIMER
    gives `X:5`, not `X:20`, at t7.
  - The outer `switch(True)` is a temporal `if` on an eval input `on: [true]`.
    A literal `true` chooses at wiring time and wires no switch
    ([control flow](control-flow.md)).

  Each child publishes in its creation cycle, because a harness literal cannot
  yet encode a member whose child output is invalid
  ([eval operator composition](eval-operator-composition.md)).

## The standard library on the base

- **Imports.** Library HGL imports core declarations and never redeclares
  them. There is no re-export.
- **Operator candidates.** The library's `impl fn` candidates for the token
  operators then implement the core identities.
  - Those that duplicate a core row add nothing once the core realises it,
    and an identical duplicate ties with the realisation (WIR-16), so the
    library drops them: the arithmetic and same-type comparison
    instantiations, and `and_`, `or_` and `not_` on `bool`.
  - A tolerant `eq_` on `f64` does not satisfy the proposed core-domain
    equality contract. Tolerance stays with `eq_epsilon`, which takes an
    explicit parameter.
  - Truthy `and_`, `or_` and `not_` on `i64`, `f64` and `str` cannot implement
    the `bool`-only core identities (§3). The library moves them to a
    separately named operator or drops them.
  - The mixed `i64`/`f64` comparisons, and a tolerant mixed `eq_`, wait on
    open point 12.
- **Library-owned declarations.** `std` declares operators outside the agreed
  core inventory. Essential named operators such as `core.abs` keep their
  declaration in `core`; `std` may extend them through additional candidates.
  Core helpers belong in `core.native`; additional standard-library helpers
  belong in `std.native`. This proposal leaves `pass_through`, `null_sink`,
  `nothing`, `const` and `tick_count` as library operators. Classification of
  the remaining named inventory follows the core inclusion criteria.
- **Provided by no layer yet:** `map_`, `switch_`, `reduce`, `mesh_`,
  `try_except`, `no_key` and `if_then_else`. Spec examples that import them
  are excluded by name from runs without the library until a layer declares
  them.
- **Higher-order `map`.** One-collection `map(...)` is a core prelude form
  (§1, §4), so the library neither declares nor re-exports it. The library
  declares no `map` and needs no change. The two-collection examples
  ([language tour](../user-guide/language-tour.md),
  [structural-types.hgl](../../examples/structural-types.hgl)) wait for open
  point 8. Until then `structural-types.hgl` keeps its
  `use std::{map}` and stays excluded by name. The explicit `map_`
  operator, with its key parameter and `no_key`, stays in no layer.
- **Tests.** Library tests that test the language or runtime, and use the
  library only for fixtures, move to the conformance pack. Tests of replay
  and record become core cases. Tests of library contracts stay with the
  library. The library's token tests move to the pack, and its tolerance
  test of `eq_` moves to `eq_epsilon`, whose default `1e-10` gives the same
  results.

## Prerequisites (doc-first)

The core's closure rests on records that are still proposed:
- [ADR 0015](decisions/0015-pull-sources.md);
- [ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md);
- [ordinary list values](ordinary-list-values.md);
- [ordinary delta types](ordinary-delta-types.md);
- [value mutability](value-mutability.md);
- [eval operator composition](eval-operator-composition.md);
- [eval recorder keys](eval-recorder-keys.md);
- [ordinary replay and record](../../../library/ordinary_replay_record.md);
- `higher-order-map.md` (new, MAP-1–MAP-7), unless MAP-1–MAP-7 stay in this
  record.

Accept them first, or accept them with this record as ADR 0017.

## Landing

1. **hgraph_spec** adds:
   - this record;
   - the prerequisite acceptances;
   - the KEYED-ROUTE-TIMER text of
     [nested graph cases](../../../runtime/cases_nested.md) in observable
     terms: "the price dictionary is bound whole to each child; its keys
     create no children". The case then names no library operator;
   - one sentence in [runtime overview](../../../runtime/overview.md),
     [node](../../../runtime/node.md) and
     [graph](../../../runtime/graph.md) ("What a nested node may do"): the
     core's language forms (§4), `map(...)` among them, are realised by
     nested nodes too, and the runtime still specifies only their components;
   - `core/`;
   - the `conformance/` skeleton and its README;
   - these amendments to accepted text, in the same change (proposed):
     - [type extensions](type-extensions.md): the expected result of a
       core-domain call binds no result parameter;
     - [runtime conformance](../../../runtime/conformance.md): the trace of a
       core-domain call is its output, not the nodes that realise it;
     - [ADR 0008](decisions/0008-temporal-contracts-and-target-mappings.md),
       Target boundary: for a core identity, operator selection is accepted
       on the call's behaviour;
     - [operators](operators.md), Signatures, lifting, and implementation:
       the 16 system identities are core contracts (§3);
     - [operator contracts](../../../library/operator_contracts.md): OP-1 and
       OP-3 move to the core.

   Leave stubs wherever another repository reads a moved file, such as
   `compiler/stdlib`.
2. **The pack module** (proposed), landed with step 1:
   - hgraph_spec adds `conformance/conformance.hgl`, the six fixture parts,
     `pack.json`, its README and the pack check. The fixtures are `export fn`
     declarations at module level of their parts.
   - Each library test file that moves to the pack changes its header to
     `module conformance part X` and gains a documentation comment.
     Each helper it copied becomes a call of the shared fixture, and a list
     constant whose type a parameter gives becomes a literal.
   - A moving test that imports a library module waits until the base layer
     supplies what it needs, such as a NaN source (open point 7).

Implementation work is tracked outside this repository.

## Rules

- **BASE-1** *(Agreed.)* A conformance test depends only on the language and
  `core`.
- **BASE-2** *(Agreed.)* The mandatory public core module is `core`. It is
  supplied without an import or a build request. Its declared implementation
  dependencies may include a selected `core.native` provider; they require no
  standard library or other application library.
- **BASE-3** The core has three surfaces: the prelude, the public interface
  and token binding.
- **BASE-4** The prelude is exactly the table in §1, resolved in the order §1
  gives.
- **BASE-5** System operators are core identities, declared as §3 gives and
  bound per target. Declaring them creates no runtime operator. The core
  fixes their behaviour over the §3 domains, not their candidate set. The
  value level is native-required. Composition is one node per call with ADR
  0008's selections and its validity requirement written out: it invalidates
  its output when an operand is invalid (OP-1) and publishes on every
  evaluation it runs. OP-3 holds. Projections that give a REF follow WIR-5,
  MAP-CONN-1–4 and TS-16 instead. Folding is value-level evaluation at compile
  time. Where a row admits several roles, they agree. Equality is exact.
  `and_`, `or_` and `not_` admit only `bool`.
- **BASE-6** A language form's observable contract is its language rule. The
  four facilities in §4 are native-required.
- **BASE-7** The six injectables are available in every admitted phase.
- **BASE-8** Eval wires the core's own replay and record and builds
  `TimedValue` lists. No module can change what eval wires.
- **BASE-9** `core/` is a reference realisation. Native substitution that
  passes the core cases conforms. A reference body is closed over the
  language, the prelude, the injectables, core declarations and their selected
  `core.native` helpers.
- **BASE-10** The test host provides everything in §6.
- **BASE-11** An import resolves only against the core and the modules the
  run supplies.
- **BASE-12** *(Agreed.)* A conformance run loads no library. A pack run
  supplies only `core`, the files `conformance/pack.json` lists for it
  and the pack's own native provider. Any selected `core.native` provider is
  part of core's declared implementation dependencies. No other run supplies
  `conformance`. The rest is proposed. A pack file imports only
  `core`, and a separate module also `conformance`, except for
  a `use` owned by an `# expect-error` and an import of another partner that
  `pack.json` lists.
- **BASE-13** *(Agreed.)* The core's conformance tests form one module,
  `conformance`, assembled from the parts `pack.json` lists. It is
  test-only: no runtime ships or links it. A fixture that two or more case
  parts or a separate module use is declared once, exported from a fixture
  part, and never copied. Rejection cases form a second module,
  `conformance.rejections`, which imports the fixtures it uses. The rest
  is proposed. Names are unique across the module, no two functions in it are
  equal up to renaming, and a case part names no other case part's
  declarations. Cases about more than one module and shadowing cases are
  modules of their own. A conformance claim runs the full assembly.
- **BASE-14** A case that needs an observation with no HGL spelling is
  host-level.
- **BASE-15** *(Agreed.)* The library builds on the base. Its HGL imports the
  core and never redeclares it. Its native parts may use runtime facilities.
- **BASE-16** *(Agreed.)* A conformance case asserts the behaviour an
  operator expression must have, never which candidate is selected. Overloads
  must not change that behaviour for the expressions the tests write.
- **BASE-17** In a conformance run, no candidate changes the behaviour of a
  core-domain call. That behaviour is its output trace (values, deltas,
  ticks, modification times and validity), its result type, its folded value,
  whether it is accepted, and a failure's catalogued code and phase. The nodes
  that realise the call are not part of that behaviour. Every use of a mapped
  token is a call of its core identity, whatever `requires` clause is active.
  The realisation has exactly one most specific candidate for every
  core-domain call, and the core descriptor names it.
- **BASE-18** The harness comparison, presence tests against `null`, `requires`
  predicates and structural projections are not core calls, and no candidate
  changes them. Outside the core domains, an implementation or library may
  accept or reject an operator expression, and the pack asserts neither. The
  exception is a result over a type that the case's own part declares with a
  concrete candidate. The rejected rows of §3 are core-domain calls and stay
  rejected.
- **BASE-19** `map(...)` is a core prelude form: the keyed child graph with a
  result. MAP-1–MAP-7 are its contract. No module re-exports it, and a module's
  own declaration named `map` shadows it there.

## Open points

Closed points keep their numbers, because the text above cites them.

1. **Core candidate set.** *Closed.* It is a lower bound: the core fixes
   behaviour over its domains, not a candidate
   set (BASE-16, BASE-17). Implementations and libraries may add candidates
   within HGL's candidate universe ([modules.md](modules.md)). Overloads a
   host registers at run time are outside that universe. An implementation
   need not hide its candidates outside the core domains. It must still
   reject the combinations the language rejects, such as `2 * cooldown` and
   `duration // duration` for tokens
   ([syntax and semantics](../developer-guide/syntax-and-semantics.md),
   Arithmetic and comparison). A conformance run loads no library (BASE-12),
   so only the realisation and the pack's own candidates reach a case (open
   point 15).
2. **replay and record as non-extensible functions.** *Closed.* Not needed
   (proposed). Eval does not choose replay and record among candidates (§5),
   so BASE-8 holds whatever a module adds. They stay ordinary, independently
   callable operators: added candidates are permitted, and a direct call
   resolves them normally
   ([eval operator composition](eval-operator-composition.md);
   [ordinary replay and record](../../../library/ordinary_replay_record.md)).
   This needs no `export fn` status.
3. **`getitem_` and `getattr_`.** The lower bound settles extensibility.
   Candidates outside the core's projection rows are allowed. One question
   remains: whether they get source declarations and join the public
   interface.
4. **Truthiness candidates.** *Closed.* They leave the core identities
   (proposed). `and_`, `or_` and `not_` constrain their operands to `bool`, so
   a candidate on another type is rejected at registration (WIR-23, WIR-24).
   As a result:
   - token and named calls agree ([operators](operators.md));
   - node evaluation, which short-circuits on `bool`, agrees with
     composition;
   - a `requires` that names these operators binds only `bool`.

   The library moves its truthy candidates to a separately named operator or
   drops them. The spec already describes `!` as Boolean negation
   ([operators](operators.md)).
5. **`len(str)`.** Whether `len` on `str` counts UTF-8 octets or Unicode
   scalar values.
6. **Windows.** `len` on windows and the window metadata join the prelude only
   if the WINDOW cases stay in the pack.
7. **A NaN source.** An IEEE-754 sentence in the `f64` domain, which depends on
   the numeric overflow policy.
8. **`map(...)` beyond MAP-1–MAP-7.** Five things are open:
   - more than one collection. Keys may align as a union, leaving a parameter
     unbound while its input lacks the key, or as an intersection. The
     two-collection examples
     ([language tour](../user-guide/language-tour.md),
     [structural-types.hgl](../../examples/structural-types.hgl)) wait for
     this;
   - a key parameter for the child;
   - list collections;
   - a named-function argument, `map(xs, child)`;
   - a `for` with a result, as sugar for `map(...)` under the same rules.
9. **Source-only eval.** An eval with an end bound would retire `consume`.
10. **Host observations.** Whether they become test-host contracts or stay
    host-level.
11. **The tolerant `f64` `eq_` test.** *Closed.* The library's tolerance test
    of `eq_` retargets to `eq_epsilon`, whose default 1e-10 gives the same
    results. The named `eq_` on `f64` is a core-domain call and must be
    exact, so the library drops its tolerant `f64` candidate. Its tolerant
    mixed candidates move to open point 12.
12. **Mixed numeric comparisons.** Comparisons between `i64` and `f64` are not
    a row. There are two choices:
    - add a row that widens and then compares exactly;
    - exclude them.

    A widening row must reconcile two texts that today read as excluding it:
    "Values of different types are not equal"
    ([scalar types](../../../runtime/scalar_types.md)) and "`1` and `1.0`
    are not equal" ([scalar cases](../../../runtime/cases_scalar.md)). Until
    this is decided, mixed comparisons are not core-domain calls and pack
    cases avoid them.
13. **Tolerant `f64` equality in a realisation.** §3 requires a core-domain
    `==` on `f64` to be exact in every role, and `!=` to be its negation. A
    realisation whose composition `==` is tolerant must become exact for
    core-domain calls without adding a second operator. A call that passes
    an explicit tolerance, such as `eq_(a, b, epsilon: e)`, is not a
    core-domain call (WIR-22) and may stay tolerant.
14. **Codes and phases for operator failures.** The `operator` category has no
    source code, and arithmetic failures have no execution code
    ([error catalogue](error-catalogue.md)). A zero divisor and `i64` overflow
    also lack a phase rule ([operators](operators.md);
    [language model](language-model.md)). Until codes exist, the pack cannot
    assert any of these:
    - ambiguity and no-match;
    - the rejected rows of §3;
    - a non-`bool` operand of `and_`, `or_` or `not_`;
    - an arithmetic failure.

    One possible shape: `operator`-category source codes for no-match and
    ambiguity, and execution codes such as `arith.zero_divisor` and
    `arith.overflow` under BYTE-3's rule, where a required constant rejects
    and optional folding keeps the execution failure. Temporal overflow in a
    constant expression stays a compile-time diagnostic
    ([syntax and semantics](../developer-guide/syntax-and-semantics.md))
    unless this point amends it.
15. **Core coherence.** *Closed for conformance isolation.* A conformance run
    loads no library, so no library candidate can change a case, and the pack
    keeps its own candidates off core rows (Pack check). A program that loads
    `std` uses ordinary matching and specificity, preserving the selected core
    operator's applicable contract; library membership gives no override
    priority. Further cross-module coherence remains outside this record.
16. **Rejection cases.** *Closed.* They form the second
    module `conformance.rejections`, which imports any fixture it needs
    from `conformance`.
17. **Exported `const fn` fixtures.** `export const fn` has no accepted
    spelling ([functions](../user-guide/functions.md)). The pack's parts share
    a `const fn` fixture without exporting it, so only a separate module that
    needs one waits for the spelling.

18. **Essential named core operators.** `core.abs` is part of the agreed core
    vocabulary. Its mandatory type domains, execution roles and boundary
    failures still need a detailed contract before conformance cases can
    claim coverage. The sixteen declarations in §3 are not an exhaustive
    core inventory.
