# User Guide

HGL is a temporal programming language for computations over values that
evolve through time. This guide describes it from an author's point of view:
how functions and types look, how change and validity affect computation, and
how source calls reach hgraph.

> This guide describes the language, independently of a compiler or runtime.
> Agreed syntax and unresolved design questions are distinguished below.
> Implementation coverage and limitations belong to the audit repository.

## Read in this order

1. [Language tour](language-tour.md) introduces a complete module.
2. [Functions](functions.md) covers composition and runtime functions,
   public exports, generic constraints and substitution, operator contracts,
   state, injectables, lifecycle, activation, and output.
3. [Types and expressions](types-and-expressions.md) defines recursive temporal
   types, nominal and generic structs, abstract data families and final
   concrete values, generic construction, sparse deltas, optional fields,
   recursive fields, rolling windows, the `atomic<T>` boundary, metadata, and runtime collection
   traversal.
4. [Modules and tools](modules-and-tools.md) covers public declarations,
   implementation discovery, native module use,
   `check`, `test`, `run`, `emit-cpp`, `hgl_add_module()`, and the REPL
   (there is no `hgl build`).
5. [Testing and running](testing-and-running.md) covers `test` declarations,
   `eval` with dense and timed sequences, expected execution errors,
   compile-rejection fixtures, running an entry from the command
   line and the REPL. Configuration-file execution remains planned.

Source examples are collected under [language/examples](../../examples/README.md).

## Current language shape

The current temporal implementation model uses `fn` and does not ask authors
to declare a `graph` or `node`. A bodyless `operator` declares a nominal,
generic callable contract whose implementations are supplied by compatible
`impl fn` definitions. Operators are public by definition; an ordinary
exact function is module-internal unless declared `export fn`.

```hgl
fn midpoint(
    tob: atomic<tuple<f64, f64>>
) -> f64 =>
    (tob[0] + tob[1]) / 2.0
```

In an ordinary `fn`, parameters are temporal by default. Parameter-level
`const` marks a wiring-time value:

Import `rolling_mean` with `use hgraph.analytics::{rolling_mean}` for this example.

```hgl
export fn smooth(
    tob: atomic<tuple<f64, f64>>,
    const window: i64
) -> f64 {
    rolling_mean(midpoint(tob), window)
}
```

Within a body, `let` introduces an immutable local and `var` introduces a
mutable local. Runtime `var` values last only for the current block execution;
persistent semantic history uses recordable `state`.

The agreed extension adds [value-level `const fn`](functions.md#value-level-functions)
for direct computations without independent ticks, and
[reconstructible caches](functions.md#reconstructible-cache) for node-local
data excluded from record/replay. See [default lifting](value-functions.md).
Non-scalar cache construction remains a separate design question.
`const fn` does not mean compile-time-only or pure; its role is distinct from
parameter-level `const`.

The current design classifies a function from the constructs used in its body:

- an ordinary expression body describes wiring composition;
- `state`, `cache`, `inject`, `start`, `when`, or `stop` makes the complete function a
  runtime implementation compiled as one node.

The agreed [iteration model](../design/iteration.md) makes `for`, `keys`,
`values`, and `items` follow the containing phase; they do not alone force a
runtime function. In composition functions, loops describe independent computations for the
elements of supported lists and maps.

Runtime functions use ordinary `return` to produce an output tick. They may
request direct output access alongside other runtime capabilities:

```hgl
fn running_total(value: f64) -> f64 {
    inject out

    when modified(value) && valid(value) {
        if valid(out) {
            out += value
        } else {
            out = value
        }
    }
}
```

This lets the same `fn` declaration either compose existing operations or
express per-tick work in one runtime function without adding `graph` and `node`
declaration keywords.

## Native boundary

Language functions call typed contracts supplied by hgraph and extension
modules. A selected native implementation may internally be a graph, compute
node, sink, source, or service. That implementation category is not part of
ordinary call syntax.

External threads, callbacks, queues, I/O, resource ownership, and push
adaptors remain C++ extension responsibilities.
