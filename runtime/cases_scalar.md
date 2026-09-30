# Scalar type cases

Status: proposed 2026-09-30, reasoned from VAL-1 to VAL-18; not yet run.
These cases observe the value layer through nodes: a node publishes what it
computes from its scalars and inputs.

## SCALAR-IDENTITY — VAL-2, VAL-3, VAL-4, VAL-5

Two types are the same type when a value of one may be used wherever a
value of the other is expected: as a node's scalar, as an element of a
set, as the value carried by a time-series bound to an input declared
with the other. Where the table says "two types", no such substitution is
admitted in either direction.

| Declaration | Result |
|---|---|
| The struct `Quote{bid: f64, ask: f64}` declared twice, in two places | one type: a `Quote` from either declaration is a `Quote` |
| `Quote` and `Level{bid: f64, ask: f64}`, same fields, different names | two types: a `Quote` is not a `Level`, and a `Level` is not a `Quote`, although their fields coincide (VAL-3) |
| A tuple `(f64, f64)` and a tuple `(f64, f64)` | one type: a tuple's identity is its positions' types |
| `Box<i64>` and `Box<f64>` | two types, and neither may stand in for the other (VAL-4) |
| `Quote` re-declared with a third field | an error, not a third type (VAL-5) |

## SCALAR-CAPABILITIES — VAL-9, VAL-10, VAL-11

| Type | equality | hash | order |
|---|---|---|---|
| `i64`, `str`, `date` | yes | yes | yes |
| an enum | yes | yes | by member integer |
| `(i64, str)` | yes | yes | yes |
| `list<f64>` | yes | yes | yes |
| `set<i64>` | yes | yes | no |
| `map<str, i64>` | yes | yes | no |
| `map<str, list<f64>>` | yes | yes | no |
| `any` | claimed | claimed | claimed |

A `set<map<str, i64>>` is a type: a map has hash. A `TSS` of a type without
hash, and a `TSD` keyed by one, are refused when the type is formed. Asking
an `any` holding a set for its order fails at that point (VAL-11).

## SCALAR-EQUALITY — VAL-7, VAL-8, VAL-12

- `{1, 2}` equals `{2, 1}` and hashes equally; `{a: 1, b: 2}` equals
  `{b: 2, a: 1}`.
- Two structs of the same type with equal fields are equal; a struct and a
  tuple with the same values are not.
- An optional field left unset equals another unset field, and orders
  before any set value.
- Values of different types are never equal: `1` and `1.0` are not equal.

## SCALAR-STRUCT — VAL-12, VAL-14, VAL-18

- Constructing `Quote` with only `bid` fails: `ask` has no default and is
  not optional. Constructing `Tagged{name: str, note: str = null}` with only
  `name` leaves `note` unset.
- An abstract `Shape` with concrete `Circle` and `Square`: a value typed
  `Shape` is exactly one of them and knows which; two `Shape` values of
  different concrete members are not equal and do not order; a `Circle`
  registered after wiring finished is unknown to a running graph.
- A tree `Node{value: i64, left: Node = null, right: Node = null}`: a value
  three levels deep copies, compares and hashes to its leaves; equality of
  two trees compares every level.

## SCALAR-NIL — VAL-1, TS-2

An invalid input read as a value is nil; a struct's unset optional field is
nil; an empty `any` is nil. Nil is one thing: a node that publishes an
`any` holding nil and a node whose output is invalid are distinguishable
only by validity, not by the value read.

## SCALAR-VIEW-AND-COPY — VAL-1, VAL-16, VAL-17

A node reads a list input in eval and keeps the view in state. In the next
cycle the producer publishes a different list. If the node kept the view it
sees the old list, because what it kept was a copy taken when it stored it;
what it reads afresh from the input is the new list. No node can change the
list through the input.

## SCALAR-TIME — VAL-15, ENG-16

- `datetime` plus a `duration` that would pass *forever* is an error, not a
  wrapped value.
- *never* plus one smallest step is *earliest start*; *latest end* plus one
  is *forever*; either plus one more is an error.
- `date` arithmetic across the representable range is an error likewise.

## SCALAR-TEXT — VAL-13, OP-9

An enum's text is its member name; a set's text lists its members in no
specified order, and two equal sets may render differently; a type held as
a value renders as its name.
