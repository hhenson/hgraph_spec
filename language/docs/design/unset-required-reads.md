# Required reads of unset ordinary observations

An admitted retained ordinary structural observation can contain unset children.
Retaining that observation or projecting a child as an observation preserves its
presence state; neither requires an absent payload to exist.

An operation that requires an ordinary payload fails with `value.unset_read`
when that observation is unset. This includes scalar arithmetic, comparisons
and Boolean conditions, and collection operations such as `len` and `items`.
An unset Boolean is not false, an unset number is not zero, and an unset
collection is not an empty collection. Even a fixed list's known size does
not make `len` of its unset observation succeed. Present false, zero and empty
values retain their ordinary behavior.

Fail before producing that operation's result or entering an iteration body.
Existing evaluation order determines which operation is reached; earlier
completed effects remain. Propagate the code through the existing execution
error contract, including graph teardown and `assert raises` matching. This
adds no evaluation order for arbitrary calls.

The rule applies to required reads, not retention, presence inspection or
structural publication. It adds no nullable type, nil literal, producer
invalidation or empty-event rule. Direct temporal-input validity requirements
and ordinary bounds and missing-key failures remain distinct; their errors
are not renamed to this code.

See [negative tests and present controls](../../examples/unset-required-reads.hgl)
and the [error catalogue](error-catalogue.md).
