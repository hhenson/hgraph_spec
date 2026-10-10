# Ordinary struct constructor argument order

Status: proposed ordinary-value construction extension, 2026-10-03.

A complete ordinary struct constructor accepts named arguments. Names select
fields; their written order selects argument evaluation order. Field
declaration order remains schema metadata and does not reorder the supplied
expressions. This matters when an argument has an observable effect or fails:
changing the order could change which effects occur before the failure.

This rule concerns ordinary struct values, including values
constructed inside a node hook. It does not define arbitrary function-call
order, temporal struct composition or contextual delta construction.

## Checking and evaluation

Before the constructor can execute, check the complete call using the existing
struct rules: resolve the concrete struct and any type arguments, associate
supplied names with fields, reject duplicate or unknown names, check argument
types and optionality, and verify required fields and defaults. An invalid
constructor is rejected before any of its argument expressions executes; an
early valid argument does not excuse a later invalid name or type.

For an admitted constructor, evaluate each supplied argument expression
exactly once, in written source order. After an expression succeeds, retain
its value independently for its named field before evaluating the next
supplied expression. Retention is recursive under
[value mutability](value-mutability.md): it does not move from the source,
keep a live mutable alias or silently widen borrowed-access permissions.
Any failure while retaining a value stops construction just as an expression
failure does. This sequencing of retention is part of the construction rule.

After all supplied arguments have succeeded and been retained, retain the
effective constant defaults for omitted fields in resolved field declaration
order. Each default is independently retained before proceeding to the next.
This is retention of an existing constant, not runtime evaluation of a new
default expression. The default uses its [defining binding scope](default-binding-scope.md),
not the supplied or already constructed instance fields. Optional fields and inherited effective defaults keep
their existing rules; an omitted unset optional field requires no payload
retention. This rule uses the resolved schema order and does not choose a new
multiple-parent field linearization.

After default retention succeeds, assemble the complete value by field
identity. Do not evaluate supplied expressions again while assembling or
arranging fields. Defaults do not add runtime argument expressions to the
written order.
The resulting value has the same nominal type and declared field identities
regardless of the order in which the caller supplied them.

For a struct declared with left followed by right:

```hgl
struct Pair {
    left: i64
    right: i64
}
```

`Pair(right: 2, left: 1)` has left equal to 1 and right equal to 2. Names
determine where values go. The expression supplied for right is evaluated
first, followed by the expression supplied for left, because that is the order
written in the call. Declaring left first does not change evaluation order.

Nested ordinary struct constructors apply the same rule recursively: an
inner constructor completes its own evaluation and retention before its
containing argument is retained and the next outer argument begins.

## Failure

If evaluating or retaining a supplied argument fails, do not evaluate later
arguments and do not produce a completed constructed value. If retaining an
omitted default fails, stop before retaining later defaults or assembling a
result. Failure during final assembly also produces no completed value. Propagate the failure through
the applicable value-operation error contract. Constant evaluation fails
checking, wiring-time evaluation fails construction, and a node hook follows
its applicable error contract; node evaluation uses the translated node error
contract.

Earlier completed effects remain. This is not a transaction over called
functions, logging, stored values or output writes. An owning assignment
whose right-hand constructor fails receives no replacement value; its
existing destination remains subject to its assignment failure contract.
No partial struct may escape as the constructor's result.

[Constructor-order cases](../../../runtime/cases_constructor_order.md) give
expected success, failure and checking observations. The
[source example](../../examples/struct-constructor-order.hgl) uses an ordinary
value helper that logs and returns its argument, solely to make evaluation
order observable in the example. Logging is not part of struct construction.
