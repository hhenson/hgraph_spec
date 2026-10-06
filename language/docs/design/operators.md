# System operators and domain properties

Status: agreed design. HGL comments use `#` or `/* ... */` (see below).

## Decisions from the discussion

- Use fixed system operator names, like a language protocol. Do not add a
  user-definable `symbol = "..."` clause or introduce Python dunder names.
- Reuse hgraph's names (`mul_`, not `mult_`) and native candidate resolution.
  Symbol syntax selects an operator identity; ordinary overload resolution
  selects its implementation and result type.
- Attach algebraic declarations to an operator **and a concrete type domain**.
  The initial vocabulary is `associative`, `commutative`, and `identity`.
- Result types and lifting are signature information, not algebraic properties.
  Division need not be closed over its input type.
- Put only semantic, public constraints on an `operator`. A dependency needed
  by one candidate's chosen algorithm belongs on that `impl fn`; it neither
  constrains sibling implementations nor becomes part of the operator API.
- No inverse declarations, group/field hierarchy, automatic reduction detection,
  or loop-to-map/reduce conversion in this iteration.
- Missing metadata means **no guarantee**. A declaration is not a proof of a
  law for every implementation or every numerical policy.

## Fixed symbol-to-name mapping

| HGL expression | System operator | Notes |
| --- | --- | --- |
| `a + b` | `add_` | Includes string concatenation where a candidate exists. |
| `a - b` | `sub_` | Binary subtraction. |
| `a * b` | `mul_` | Multiplication. |
| `a / b` | `div_` | Numeric true division; `i64 / i64` produces `f64`. |
| `a // b` | `floordiv_` | Floor quotient; `i64 // i64` produces `i64`. |
| `a % b` | `mod_` | Floor-based modulo, not C++ truncating remainder. |
| `-a` | `neg_` | Unary negation is a separate operator. |
| `!a` | `not_` | Boolean negation. |
| `a == b` | `eq_` | Equality. |
| `a != b` | `ne_` | Inequality. |
| `a < b` | `lt_` | Ordering requires an applicable candidate. |
| `a <= b` | `le_` | Ordering. |
| `a > b` | `gt_` | Ordering. |
| `a >= b` | `ge_` | Ordering. |
| `a && b` | `and_` | Graph composition wires both operands. |
| `a \|\| b` | `or_` | Graph composition wires both operands. |
| `a[index]` | `getitem_` | Existing collection projection rules also apply. |
| `a.field` | `getattr_` | Existing structural field projection rules also apply. |

`+=`, `-=`, `*=`, and `/=` combine assignment with the corresponding binary
operation; they do not introduce four more operator identities. Assignment and
comparison are distinct. Node Boolean expressions use short-circuit
evaluation; graph Boolean expressions do not conditionally wire their RHS.

Floor division maps `a // b` to the fixed `floordiv_` system identity, including
`i64 // i64 -> i64`. HGL uses `#` for line comments and `/* ... */` for block
comments, so the operator is unambiguous in every expression context. Power,
bitwise, shifts, unary plus, and user-defined tokens have no new HGL syntax in
this iteration; their native named operators remain separate library inventory
work.

A local function named `add_` does not change `a + b`. Explicit named calls
still follow normal local/import lookup. System symbols resolve the native
identity, not the nearest function with a matching short name.

## Execution role and native implementations

Status: agreed direction. The existing symbol mapping and
domain-property syntax are unchanged.

An operator may have both temporal and value-level implementations, with
either role implemented in HGL or natively. A graph-construction call can
select a temporal candidate to compose or wire; a call inside node evaluation
requires a value-level candidate. A permitted wiring-time value call may also
use a value-level candidate. Role, concrete signature/domain, and constraints
determine eligibility; native implementation language alone does not.

`const fn` is the agreed value-function marker. Its combination with operator
implementation and native declaration syntax remains open. Selection must
preserve the nominal operator and shared matching rules, without adding
per-tick overload lookup or a second language-local dispatcher. A value helper
does not silently become a temporal node: lifting needs an explicit contract
for activation, validity, REF access, and output/deltas as well as types.

Algebraic properties still require the precise domain, selected candidate,
and numerical policy before a transformation is legal. Merely supplying both
execution roles does not prove they satisfy identical laws. See
[ADR 0008](decisions/0008-temporal-contracts-and-target-mappings.md#operators-and-native-implementations)
for the full distinction and compatibility boundary with existing `native fn`.

## Domain-bound declarations

```hgl
operator mul_<T>(lhs: T, rhs: T) -> T
    properties<i64> { commutative, identity = 1 }

operator add_<T>(lhs: T, rhs: T) -> T
    properties<str> { associative, identity = "" }
```

`properties<...>` belongs to the preceding operator. Indent its clauses and
leave a blank line before the next definition; see [Formatting](formatting.md).

`properties<...>` follows the signature and optional `requires` clause. Entries
are comma-separated; a trailing comma and multiline layout are allowed.
Multiple clauses specialize the **same nominal operator**, not unrelated
operators that happen to have its short name. Properties apply to conforming
candidates in that domain; a consumer must still identify the actual selected
candidate and its numerical policy before using a property as a rewrite rule.

The selector binds declaration type parameters **in declaration order**:

```hgl
operator mul_<L, R, O>(lhs: L, rhs: R) -> O
    properties<i64, i64, i64> { commutative, identity = 1 }
```

This selects `(i64, i64) -> i64`, not every candidate containing an `i64`.
It says nothing about `(i64, f64) -> f64`. The domain must satisfy the operator's
`requires` clause. Unknown properties, repeated properties, repeated domains,
wrong selector arity, non-concrete types, and incorrectly typed identities are
errors. The current property syntax admits concrete type selectors and scalar
constant identities; const-generic selectors, partial domains, and value-range
or numerical-policy predicates are deferred. `ref` and `signal` are not value
domains for these algebraic declarations.

Both laws describe binary operators with the same two input types.
Associativity and identity additionally require closure, `(T, T) -> T`.
These are fixed binary signatures: a parameter pack is not one scalar operand,
and property domains cannot bind type packs.
Commutativity may describe `(T, T) -> bool`, for example equality. An identity
is two-sided and must be a compile-time value of the result type. An identity
alone proves neither associativity nor commutativity.

## Numerical exceptions and reduction

Algebraic laws describe observable behavior, including failure and special
values, not just a mathematical analogy:

- String concatenation is associative but not commutative; `""` is its identity.
- Signed integer addition/multiplication cannot advertise unconditional
  associativity without an overflow policy. Regrouping can change which
  intermediate operation overflows.
- Floating-point addition/multiplication are not exactly associative.
  NaNs, signed zero, infinities, rounding, and exception policy also matter to
  identities and reordering. There is no unconditional floating-point
  associativity declaration or implicit “fast math” permission.
- Floating `min`/`max` cannot inherit total-order laws when NaNs and signed-zero
  selection are observable. Custom native/Python scalar operators inherit no
  laws merely because C++ supplies an overloaded operator.

Scalar comparisons follow [NaN comparisons](nan-comparisons.md), including
self-inequality and unordered comparisons when either operand is NaN.

An implementation may attach verified, specialization-specific laws to native
kernels. Unknown guarantees remain absent. Such metadata does not change the
source type or its arithmetic policy.

Property declarations survive checking and module import. Checking validates
the metadata shape and identity value against the substituted result type,
including ordinary `i64` to `f64` widening; it does not prove a mathematical
law. Declarations alone neither establish trusted kernel guarantees nor
authorize optimization. Proof transport remains open.

An explicitly requested `reduce` retains its own contract. Removing an unsafe
native law prevents the lifted reduction fast path; it does not silently change
the caller's chosen tree reduction into an ordered fold. Reduction `zero`
remains distinct from a kernel identity: no implicit substitution, especially
for empty or singleton inputs. Unordered maps need an appropriate reduction;
order-sensitive lists may need linear reduction. Automatic dynamic-loop
accumulation remains deferred as agreed in [Iteration](iteration.md).

## Signatures, lifting, and implementation

```hgl
operator div_<L, R, O>(lhs: L, rhs: R) -> O
```

This contract admits a candidate `(i64, i64) -> f64`. A floor-division candidate
can instead be `(i64, i64) -> i64`. Selecting an output type is ordinary
candidate resolution, not an `inverse`, `associative`, or “loss” annotation.
Floor division rounds down (`-7` divided by `3` yields `-3`), and modulo has the
corresponding sign (`-7 % 3 == 2`, `7 % -3 == -2`). Division by zero is an error
for these default operations. Named native calls may expose explicit policies.
Floating-point modulo forms a remainder directly, then adjusts to the divisor's
sign (including signed zero). It does not form `lhs / rhs`: an overflowing or
underflowing quotient must not corrupt the remainder. For example,
`1.0 % (1e308 * 2.0)` is `1.0`, even though the divisor is positive infinity.
Constant folding, graph wiring, and node evaluation follow this same rule.

Native scalar lifting wraps a precise function signature as a time-series
candidate. A graph expression wires that candidate; node code evaluates its
scalar operation. See the [paired HGL/C++ examples](https://github.com/hhenson/hgraph/blob/main/language/docs/developer-guide/operator-cpp-mappings.md).

The executable [operator module](https://github.com/hhenson/hgraph_std/blob/main/hgl/hgraph/operators.hgl) declares
the 16 arithmetic/comparison/Boolean hooks (including floor division),
delegates implementations to production native candidates, and materializes
the supported primitive combinations. The native-call requirements therefore
belong to those delegating `impl fn` candidates, not the public contracts. Its
`hgraph.operators.*` identities are parallel migration contracts, not
replacements for the native system identities.
The existing `getitem_`/`getattr_` projections are not redeclared as scalar binary
arithmetic. Broader temporal, collection, and downstream scalar domains remain
in the native registry until their HGL materializations are covered.

## Native scalar requirements

A value dependency names its native scalar function, not a temporal operator
being replaced. `requires native::add(L, R) -> O` requires exactly one matching
`native const fn` overload with value parameters. Types match exactly; no
numeric widening, input views or temporal candidates satisfy it. A unique
result can bind `O`; missing and ambiguous overloads reject the candidate.

The generic body calls `native::add(lhs, rhs)`. Its requirement justifies that
call while types are symbolic; specialization selects the concrete native
entry before execution. It does not justify `lhs + rhs`, establish an algebraic
law, or recursively ask the temporal `add_` implementation to validate itself.
Native phase restrictions and capability propagation still apply. A requirement
alone executes nothing and adds no graph node.

Validation cases: integer addition returns `i64`; mixed integer/float addition
returns `f64`; string concatenation returns `str`. Wrong result types, absent
overloads, temporal functions and ambiguous signatures are rejected. Renaming
the native helper does not affect eligibility. Imported descriptors preserve
this value dependency and require its native interface to be available.
