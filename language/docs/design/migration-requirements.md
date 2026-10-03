# Standard-library migration requirements

Status: language requirements identified during library extraction. MIG
identifiers remain stable; implementation progress belongs to the audit catalogue.

The [catalogue](https://github.com/hhenson/hgraph_spec_audit/blob/main/catalogue/README.md) answers *which operator is
blocked on what* with six blocker families (B1–B6). This ledger answers the
question one level up: which language or runtime contract each family is
waiting on, what part of it is already accepted, and which decision is still
open. A requirement is not a feature request; the catalogue does not authorize
inventing syntax for a blocked entry, and neither does this page.

## Two identifier families

- `HGL-MIG-001` … `015` name language, runtime and packaging contracts that the
  first library extraction discovered. They were numbered in PR #801 and are
  kept stable so catalogue reviews, ADRs and source comments can cite them.
- `HGL-LIB-001` … `004` name the parity blockers recorded beside the compiled
  `standard.hgl` slice. They are narrower than a `MIG` entry and each maps to
  one.

## Requirements at a glance

| Requirement | Contract |
| --- | --- |
| MIG-001 module parts | One module identity across source files (ADR 0006) |
| MIG-002 parameter packs | Pack binding, cardinality and reflection (ADR 0007) |
| MIG-003 algebraic properties | Domain-scoped claims, not automatic optimization permission |
| MIG-004 scalar/native boundary | Value, borrowing, ownership and error contracts |
| MIG-005 recordable state | Authoritative state distinct from reconstructible caches |
| MIG-006 collection mutation | Typed writes preserving temporal delta rules |
| MIG-007 delta forwarding | Type-preserving transfer of complete deltas, including removals |
| MIG-008 output resolution | Dependent schemas obey shared resolution rules |
| MIG-009 operator identity | Source naming preserves canonical operator identity |
| MIG-010 implementation arity | Candidate refinement, extra parameters and ranking |
| MIG-011 higher-order forms | Branch and child-graph ownership, inputs and results |
| MIG-012 effects and capabilities | Explicit capability, phase, lifetime and effect requirements |
| MIG-013 library metadata | Documentation survives checking and emission |
| MIG-014 empty input policies | Explicit activation and validity selection |
| MIG-015 generic publication | Materialization and retained resolver parameters |

LIB identifiers and measured migration coverage are maintained in the
[audit catalogue](https://github.com/hhenson/hgraph_spec_audit/blob/main/catalogue/README.md).

## Open decisions, by requirement

Only entries with a decision not recorded elsewhere are expanded here.

### MIG-005: generic recordable state

`dedup`, `take` and `drop` fit the existing scalar `state` form. Other stream
nodes need generic state with no natural default, sparse validity, queues or
windows. The decision is between optional state cells, constructor functions,
and an admitted opaque native state type. Whatever is chosen: state that
affects later output remains recordable, and scratch caches must not be
disguised as recordable state (ADR 0008 fixes that distinction).

### MIG-007: delta capture and forwarding

`pass_through<T>` applies the input's delta to an independent output of the
same temporal shape. The accessor is `delta_value(value)`, as specified by
the [delta-value contract](delta-value-metadata.md), including the generic
runtime body. Its scalar result and admission are defined for the eight
scalar domains. Structural and collection delta types and output application,
including removals, need a separate contextual contract; replacing them with
complete value snapshots does not satisfy this requirement.

`delta<S>(...)` is the distinct sparse-update constructor. It is not an
accessor. The [ordinary delta type contract](ordinary-delta-types.md) supplies
`delta_of(T)` as its storable source type for admitted shapes.

### MIG-008: output and type resolution

Whatever expresses dependent outputs for `convert`, `combine`, `collect`,
`split`, frame joins and higher-order calls must obey the shared resolution
rules. A compiler-specific ranking or inference policy is not acceptable, and an unconstrained generic `O` on a contract is not a resolver.

### MIG-009: source names versus native identities

A source name must retain its canonical operator identity across native
bindings. Renaming a target-language symbol must never create a new overload
family. Compiler-internal helper names are not public HGL declarations.

The library constant source is named `const`, never `const_` (owner ruling,
2026-09-29). It has an independent scalar input type and temporal output shape;
the output is resolved from the value or explicitly selected, and `delay`
follows the type selection. A `const(f)` call whose argument names a function
is the value-role selector; other `const(...)` calls resolve to the operator
in scope. The [bootstrap](const-debug-bootstrap.md) uses a private `const_`
helper and does not rename the library operator.

Likewise, a native binding must preserve an operator's public parameter names,
defaults and type relationships, including named-call behavior.

### MIG-010: fixed candidates beside pack contracts

Native candidates may refine a variadic contract with a fixed arity or add
implementation-specific scalar parameters. Undefined: how an `impl fn` declares
that relationship, how a call discovers the extra parameters, and how ranking
compares fixed and packed forms. Algorithm dependencies stay on the candidate
and never leak to sibling implementations.

### MIG-012: effects and capabilities

Source-native evaluation functions remain non-blocking. ADR 0009 admits
`throws` and the descriptor's `translated` policy under hgraph's node error
model; functions without `throws` remain `noexcept`. Owned scalar results are
admitted, while owned non-scalar results remain B1. ADR 0010 specifies the
clock and scheduler capabilities, scheduled activation, and evaluation-time
input activity. Non-recordable storage, input access in lifecycle hooks, and
external resource ownership remain B2. A C++ body is not permission to publish
a contract the descriptor cannot enforce: the vocabulary must be closed and
shared between HGL source, descriptors and the backend-neutral runtime
specification.

### MIG-013: library documentation and compatibility metadata

Structured [source documentation](documentation.md) is agreed: declaration-attached
Google-style sections with reST content survive checking, lowering and emission.
Comments remain non-executable. Defaults, stability and compatibility metadata
remain open.

### MIG-015: open generic implementation publication

`instantiate op<A, _>` closes some positions and retains others for the
resolver. Collection length needs no body-visible reification because the
native view reads live metadata. Where a body genuinely reads the selected
value, the design must choose the owner of later materializations: the
consuming AOT module, a descriptor-backed implementation factory, or an
explicit body-availability contract on a retained candidate. The choice must
keep one operator identity, ordinary ranking, module lifecycle removal,
readable generated code and backend portability.

## Family design questions

- Comparison results and civil-time policy parameters need nominal enum
  contracts rather than representation-dependent integers.
- One nominal operator may span domains and call shapes without source-order
  dispatch.
- Set operations require membership and simultaneous-delta rules.
- Ordered handlers, reference reselection and source activation must retain
  their own temporal contracts when library bodies are re-expressed in HGL.

## Definition of migrated

Migration coverage is an audit result, not a language rule. A replacement
must preserve the public contract, provider identity and observable behavior;
acceptance requires validated scenarios for the admitted domain.
