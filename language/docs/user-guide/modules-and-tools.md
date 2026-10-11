# Modules and tools

Modules make hgraph and extension capabilities available without exposing a
general native foreign-function interface.

## Declaring a module

Every source file begins with one module declaration:

```hgl
module examples.prices
```

The module name is canonical and dot-separated. A large module can split its
declarations into explicitly named parts:

```hgl
# arithmetic.hgl
module examples.prices part arithmetic

fn midpoint(tob: atomic<tuple<f64, f64>>) -> f64 =>
    (tob[0] + tob[1]) / 2.0
```

```hgl
# smoothing.hgl
module examples.prices part smoothing

use hgraph.analytics::{rolling_mean}

export fn smooth(
    tob: atomic<tuple<f64, f64>>,
    const window: i64,
) -> f64 => rolling_mean(midpoint(tob), window)
```

All files in a multi-file compilation declare the same module and a unique
`part` name. They share one declaration scope, so a private helper such as
`midpoint` is visible in another part. They also produce one public module,
public namespace; a part never adds
an import path or re-exports a declaration. Part names provide deterministic
compiler ordering only, and moving a declaration between parts does not change
its identity. Each file keeps its own rule that `use` declarations precede
ordinary declarations.

The compiler does not discover sibling files. The command line or build target
must list the complete part set explicitly. A single file may carry a `part`
label while being checked in isolation; the label matters only when files are
assembled.

## Importing declarations

`use` can import an explicit set of public declarations into the local scope:

```hgl
use hgraph.std::{if_then_else}
use hgraph.analytics::{rolling_mean, zscore}
```

Alternatively, a module alias creates a namespace without adding its
declarations as unqualified names:

```hgl
use my.analytics as analytics

analytics::zscore(prices)
```

An `operator` declaration is public automatically. An ordinary exact function
is public only when declared `export fn`, and a nominal struct only when
declared `export struct` or `export abstract struct`; unexported functions and
structs remain module-internal. Both selective and aliased imports expose only
this public declaration surface.

There are no wildcard imports or re-exports. In particular, an implementation
module does not create a second import path for an operator defined elsewhere.
Canonical scalar and container types, temporal-shape rules, `atomic<T>`, and
expression operator bindings come from the implicit hgraph prelude. Exported
nominal structs and their abstract-family relationships follow their defining
modules and are imported like other public declarations. Everything else must
be imported, reached through a module alias, or declared locally.

An exported generic struct exposes one nominal family, not a separate export
for every possible specialization. Downstream code applies that family with a
complete argument list such as `Box<f64>`.

## Native functions

Call an imported native function using its HGL signature, just like another
module function. Its declared parameter types and permitted call phases apply.
A fallible helper can end the current evaluation; writes already made in that
evaluation are not rolled back. HGL has no `try` statement.

Native implementation parts obey the [shared binding contract](../design/native-interfaces.md).

## Core utility functions

The `hgraph.native` module provides `len` and `is_empty` for `str`,
fixed and unbounded lists, sets, maps, and tick-count rolling windows. It also provides `valid`, `all_valid`, `modified`,
`last_modified`, `bound`, and `active` for every standard time-series shape.
Additional string and window queries live in parts of the same module; see
the [native inventory](https://github.com/hhenson/hgraph/blob/main/language/stdlib/hgl/hgraph/README.md). Value operations keep
ordinary HGL spelling rather than exposing the underlying representation.
An HGL library imports the module normally:

```hgl
use hgraph.native as native

fn list_size<T, const size: i64>(value: list<T, size>) -> i64 {
    when {
        return native::len(value)
    }
}
```

The native library must be included in the package that runs the program.
For build configuration, see the
[package-authoring guide](https://github.com/hhenson/hgraph/blob/main/language/docs/developer-guide/native-modules-and-packages.md#building-a-package).
The [module inventory](https://github.com/hhenson/hgraph/blob/main/language/stdlib/hgl/hgraph/README.md) lists available
functions and current type restrictions.

## Operator identity and implementation binding

An operator is identified by its defining module and name, not by its short
name alone. The qualified operator names `market.pricing.value` and
`risk.pricing.value` therefore denote distinct contracts. Canonical identities
appear in diagnostics and metadata; source calls qualify through a local module
alias rather than spelling a dotted module path as an expression.

An implementation module binds an `impl fn` to a local operator declaration
or to exactly one operator brought into local scope by a selective import:

```hgl
module market.pricing_impl

use market.pricing::{value}

impl fn value(input: f64) -> f64 =>
    input
```

The `impl` modifier is what creates the binding. Without it, `fn value` beside
an in-scope operator `value` is a name conflict; with it and no operator in
scope, the declaration is an error. Adding a `use` for an unrelated call can
therefore never turn one of your helpers into a published candidate.

The uniqueness rule applies per local short name. Selectively importing two
different operator definitions as `value` is an import error before function
checking. A module alias does not establish an implementation binding, so an
implementation module may still use other same-named operators explicitly:

```hgl
module market.pricing_impl

use market.pricing::{value}
use risk.pricing as risk

impl fn value(input: f64) -> f64 =>
    input

fn compare(input: f64) -> bool =>
    value(input) == risk::value(input)
```

At a call site, name resolution selects the nominal operator first. Hgraph's
resolver then considers only implementations registered for that operator.
Namespace qualification therefore resolves collisions between different
operator definitions; it does not break a tie between implementations of one
operator. Equal-ranked implementations within one selected operator remain an
ambiguity error.

Every non-generic `impl fn` contributes an implementation candidate. A generic
`impl fn` contributes only the candidates requested by `instantiate`; `_` may
retain selected resolver slots, but the unrestricted source template is not a
candidate. Neither form uses `export` or is separately
importable by its implementation module's name. `export fn` is reserved for
exposing an ordinary exact function.

### Compiling a separate implementation

A separately compiled contract retains its identity, signature, requirements
and properties when imported. Implementation-specific requirements remain on
the candidate. If an implementation cannot preserve an imported contract, it
must report that limitation rather than approximate the contract.

For example, compile this contract independently as `contracts.hgl`:

```hgl
module example.contracts
operator adjust<T>(value: T, const amount: T) -> T
```

Then compile `provider.hgl` against its descriptor:

```hgl
module example.provider
use example.contracts::{adjust}

impl fn adjust<T>(value: T, const amount: T) -> T
requires T in {i64, f64} {
    when { return value + amount }
}
instantiate adjust<i64>, adjust<f64>
```

A consumer imports `example.contracts` to name `adjust`. Its application must
also enable a package providing a compatible implementation. The contract and
implementation can be authored in separate modules; importing the contract
alone does not supply its implementation. See the
[package-authoring guide](https://github.com/hhenson/hgraph/blob/main/language/docs/developer-guide/native-modules-and-packages.md#building-a-package)
for linking and registration.

## Implementation discovery

Imports determine which names your source can use. The application determines
which implementation packages are enabled. Installing a package somewhere on
the machine does not automatically make its implementations available.

Operator resolution considers enabled implementations of the selected contract.
It does not choose a different contract just because the names look alike.
Current builds require explicit dependencies; automatic discovery and locking
of a complete transitive package set remain planned.

## Native adaptors stay native

A C++ extension may expose a typed function contract backed by a graph, node,
source, sink, service, or operator overload:

```hgl
use acme.market_data::{subscribe_quotes}
```

The language call can supply temporal arguments and `const` configuration
declared by that module. The C++ package remains responsible for callback
admission, threads, queues, backpressure, resource ownership, start and stop,
protocol acknowledgement, and teardown.

Language source cannot declare an adaptor. A top-level `native fn` may contain
local evaluation-time C++, but it does not acquire callback, thread, queue,
service, module-lifecycle, or external dependency semantics.

## Command-line workflow

A toolchain can expose separate checking, testing, execution and emission
commands. Command names, flags and generated filenames are tooling choices;
they do not alter the language rules.

## Building a package

A package supplies its module interfaces and selected native dependencies.
Importing a contract does not install or load an implementation. The build
must make the chosen provider available before wiring.

## Execution requirements

Interpreted and compiled execution obey the same language and runtime rules.
Compiler, linker, cache and platform requirements are implementation-specific.
