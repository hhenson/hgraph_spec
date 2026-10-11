# Bytes values and publications

`bytes` is an atomic scalar value containing a finite sequence of uninterpreted
octets. Its storage is not an HGL list or structural time-series shape.

## Rules

- **BYTE-1 — Construction.** `bytes()` constructs a present empty value.
  `bytes(octets)` takes one ordinary `list<i64>` or fixed `list<i64, N>` and
  constructs a value with those octets in order. Each integer must be in the
  inclusive range 0 through 255. Arguments are positional. These are the only
  constructor forms; wrong
  argument count or type fails checking. There is no integer-length or text
  encoding overload, implicit conversion, or byte-string literal.
- **BYTE-2 — Argument context and phase.** The constructor executes in its
  call's ordinary evaluation phase, including a value-function call or a
  readable value expression inside node evaluation. It does not read a
  temporal port during graph wiring or introduce an implicit temporal lift.
  Evaluate the ordinary list argument exactly once before checking its octets.
  Existing ordinary list-literal admission applies; `bytes([])` supplies
  expected type `list<i64>`. No runtime-expression list-literal extension is
  introduced.
- **BYTE-3 — Range failure.** After its argument succeeds, the constructor
  checks every octet. An out-of-range integer fails with execution code
  `value.byte_range`; no partial bytes value escapes and no truncation or
  wrapping occurs. A failure during required constant evaluation rejects that
  constant. A constructor in an executed test or runtime body retains its
  execution failure, even when its arguments are constant; optional folding
  must not turn that failure into source rejection. Existing effects before
  failure stand. Argument failures retain their own errors.
- **BYTE-4 — Value operations.** Equality compares exact octet contents and
  length. Ordering is unsigned lexicographic order: compare the first differing
  octet as an integer from 0 through 255; a proper prefix precedes the longer
  sequence. Bytes has equality, hash and order capabilities; equal contents
  have equal hashes. `len(value)` on an ordinary bytes value returns its octet
  count as nonnegative i64. Admitted counts must fit i64. This extension adds
  no byte indexing, slicing, mutation, text encoding, decoding or prescribed
  text/hash representation.
- **BYTE-5 — Ownership and identity.** Construction independently retains the
  octets; later changes to the source list cannot change the result. Consumer
  access and every owning retention boundary obey VAL-16/17 and TS-22.
  Copies preserve exact contents and canonical type. `atomic<bytes>` is
  `bytes`, and `delta<bytes>` is exactly `bytes`; construct deltas with ordinary
  bytes values, without a scalar delta-constructor overload.
- **BYTE-6 — Publications.** Admit bytes leaves recursively in the finite
  structural publication profile, ordinary atomic payloads and rolling
  arrivals. Admit bytes as set members and map keys under the existing exact
  equality/hash and constant-key rules. Use guarded `delta_value`, generic
  pass-through, `TimedValue<T>`, replay and record unchanged. Present empty and
  equal bytes values publish ticks; `_` is silence. Sparse omissions remain
  omissions, atomic payloads remain complete, and rolling deltas remain
  arrivals. Constructors in supplied eval arguments execute before target
  start; constructors in the target execute in their ordinary call phase.
  Harness comparison uses BYTE-4 for present byte leaves. A constructor range failure retains
  `value.byte_range`, rather than becoming an input-profile error.

## Examples and acceptance

```hgl
let empty: bytes = bytes()
let octets: bytes = bytes([0, 255])
assert len(empty) == 0
assert len(octets) == 2
assert octets == bytes([0, 255])
assert bytes([0]) < octets
assert bytes([127]) < bytes([128])
```

[The source example](../../examples/bytes-values.hgl) covers present empty,
equal and distinct publications, silence and runtime range failure.
[Required cases](../../../compiler/cases_bytes_values.md) include ownership,
recursive admission and ordinary replay/record. The
[ordinary delta](ordinary-delta-types.md),
[atomic equivalence](atomic-scalar-equivalence.md),
[scalar key](scalar-collection-keys.md) and
[rolling arrival](rolling-publications.md) contracts retain their existing
rules. Any values, native atomics and REF designation are separate extensions.
