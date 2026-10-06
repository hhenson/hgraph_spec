# NaN scalar comparisons

For `f64` operands, if either operand is NaN, `==`, `<`, `<=`, `>` and `>=`
return `false`; `!=` returns `true`. This includes comparing a NaN with itself
and applies in constant evaluation, graph construction and node evaluation.
Comparison neither raises nor changes an operand. Finite comparison rules,
including an operator's explicit tolerance parameters, remain unchanged.

An optimizer must preserve these results: it cannot replace `x == x` with
`true`, `x != x` with `false`, or an ordered comparison with the negation of
its opposite unless it has proved the relevant operands cannot be NaN.

Existing logarithm rules provide a NaN value for a negative `f64` operand.
A test can observe its classification through an ordinary Boolean result:

```hgl
# A user-defined observer, not a new built-in operation.
fn is_nan(value: f64) -> bool {
    when { return value != value }
}
```

NaN remains a present scalar publication. Replay, `delta_value`, pass-through
and recording must retain its NaN classification and publication time; equal
bit patterns are not required. Test the Boolean observation rather than
asserting equality of NaN payloads. This adds no NaN literal, constructor,
signalling-NaN contract, payload/sign guarantee, or collection-key equality
policy. The [NaN key boundary](scalar-collection-keys.md) remains open.
