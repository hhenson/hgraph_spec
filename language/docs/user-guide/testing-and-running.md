# Testing and running

This page describes tests, evaluation and run configuration. Provisional
forms are labelled. The language rules are in
[Tests and the evaluation harness](../developer-guide/syntax-and-semantics.md#tests-and-the-evaluation-harness)
and [Running a module](../developer-guide/syntax-and-semantics.md#running-a-module).

A module carries its own tests, and a module is run from outside its source.
This page shows both from the author's side.

## Tests live beside the code

A `test` declaration is a named block at module scope. It sees everything
its module declares, exported or not, so a private helper can be tested
without exporting it:

```hgl
module examples.midpoint

fn midpoint(tob: atomic<tuple<f64, f64>>) -> f64 =>
    (tob[0] + tob[1]) / 2.0

test midpoint_ticks {
    assert eval(midpoint, tob: [(1.0, 2.0), (2.0, 3.0)]) == [1.5, 2.5]
}
```

`hgl test path/to/program.hgl` runs every test in the module and reports
each failing assertion with the cycle at which the expected and observed
outputs first differ:

```text
midpoint_ticks ... ok
1 test, 0 failed
```

Test names after the file select a subset. Tests are never part of a built
artifact.

## Test-only helpers and module parts

Wrap test helpers and their cases in an unnamed `test { ... }` context:

```hgl
module examples.test_contexts

export fn identity(value: i64) -> i64 => value

test {
    fn fixture(value: i64) -> i64 {
        when { return value + 1 }
    }

    test fixture_ticks {
        assert eval(fixture, value: [1, 2]) == [2, 3]
    }
}

test {
    test shared_fixture {
        assert eval(fixture, value: [3]) == [4]
        assert eval(identity, value: [3]) == [3]
    }
}
```

All test contexts contribute to **one module-wide test scope**. Helpers are
visible in every test context and named test in that module, including across
its parts and regardless of declaration order. A context groups source code;
it does not create a separate namespace. Test code can use the module's
ordinary private declarations too.

Production functions cannot reference these helpers. Other modules cannot
import them, and helpers cannot be exported. A test helper may shadow a
production function; the production definition is unaffected. Within each
scope, the ordinary const/temporal function selection rules still apply.

A test context contains private `fn` and `const fn` declarations and
named `test` cases. Put assertions inside a named case, not directly in the
context. Nested contexts, test-local types, imports, native declarations, and
operator implementations are not supported in this first slice.

`hgl test` includes and runs the helpers; `emit-cpp` and `hgl_add_module()` omit
test-only helpers and cases from the package. In the REPL, a context adds its helpers to
the session and runs only its newly declared cases. A helper-only context
does not rerun earlier tests.

When splitting a module into parts, place helper declarations in any part's
test context and pass that part alongside the cases:

```sh
hgl test api.hgl --part helpers.hgl --part cases.hgl
```

## Assertions and evaluation

`assert` takes any wiring-time `bool` expression. `eval` drives a function
by wiring replay source operators, the function, and a record sink operator.
Its first argument is a module `fn`; the rest bind to that function's
parameters exactly as a call would,
positionally or by name. To evaluate an operator, wrap it in a `fn`.

Eval configures these operators and owns the run's typed input and capture
buffers; the test does not call or configure replay and record separately.
They remain normal operators that may also have independent library APIs.
Eval chooses recorder keys before start that differ from every resolved
source, seed and other internal key in the run, including known nested graph
requirements. Source-selected keys keep their ordinary sharing semantics;
see [recorder-key ownership](../design/eval-recorder-keys.md).
The recorder retains copies of output deltas, so later ticks cannot change
an earlier result. The [operator-composition contract](../design/eval-operator-composition.md)
describes this boundary.

A local `const fn` can also be evaluated through its default lifted node.
`eval(const(scale), value: [1.0, 2.0], factor: 3.0)` explicitly selects the
value definition even if a temporal `scale` exists. Without `const(...)`,
the temporal definition wins. See [Value functions and lifting](value-functions.md)
for mixed scalar/temporal arguments and direct all-scalar assertions.

## Expected execution errors

Use a listed [execution-error code](../design/error-catalogue.md#execution-errors)
with `assert raises` inside a named test:

```hgl
fn negative_delay() -> i64 { yield -1us: 1 }
fn consume(tick: i64) { when { } }
fn drive_negative(tick: i64) -> i64 {
    consume(tick)
    negative_delay()
}

test rejects_negative_delay {
    assert raises("yield.negative_duration") {
        eval(drive_negative, tick: [1, 1])
    }
    assert true
}
```

The block runs once. A matching error passes after cleanup, then the test
continues. Normal completion, another error, an assertion failure or failed
cleanup fails the test. The code must be a string literal from the catalogue;
messages are not matched. Graph effects completed before failure remain.
Errors a graph publishes as data use ordinary output assertions. See the
[complete rules](../design/execution-error-assertions.md) and
[examples](../../examples/execution-errors.hgl).

## Expected source errors

Keep intentionally invalid source in a separate rejection fixture:

```hgl
module examples.bad_window

# expect-error(type, "rolling.size_kind")
fn consume(value: rolling<f64, 5m, 3>) { when { } }
```

Run `hgl test --reject fixture.hgl`. The annotation expects exactly one
primary error on the immediately following physical line in that file.
Every primary error must be expected, with an exact category and code;
message text is not matched. Unknown names, wrong locations, extra errors
and successful compilation fail the fixture. Warnings and attached notes
are not matching errors. `build` is not an eligible rejection category.

The [catalogue](../design/error-catalogue.md) enumerates all categories and
allowed source-error codes. The category name is annotation syntax, not a
bare identifier value in an HGL expression. See the
[fixture rules](../design/compile-rejection-fixtures.md) and
[isolated examples](../../examples/reject/README.md).

## Dense sequences

A temporal parameter receives a *sequence*: one element per engine cycle,
starting at the first cycle of the run. `_` means the input does not tick in
that cycle. The result of `eval` is the output's sequence, with `_` where
the output did not tick:

```hgl
test midpoint_skips_silent_cycles {
    assert eval(midpoint, tob: [(1.0, 3.0), _, (2.0, 4.0)]) == [2.0, _, 3.0]
}
```

Cycle 1 has no input, so `midpoint` does not tick and the expected sequence
says `_` there. An element has the shape of the parameter's value type: a
scalar for `f64`, a tuple for `atomic<tuple<f64, f64>>`. Because the tuple
is `atomic`, it arrives whole and `_` stands for the whole element. The
observed sequence runs through the later of the last input cycle and the
last output tick, and `==` requires the same length and equal elements.

Silent cells count toward the input horizon even when no output is produced.
For a scalar runtime pass-through, all-silent input `[_, _, _]` therefore
compares with `[_, _, _]`, and empty input `[]` compares with `[]`. These
runs require an empty recording, as OP-11 specifies. A failed run or missing
recording is not an empty successful result.

A `const` parameter receives a constant, not a sequence:

```hgl
fn scale(value: f64, const factor: f64) -> f64 => value * factor

test scale_applies_factor {
    assert eval(scale, value: [1.0, _, 2.0], factor: 3.0) == [3.0, _, 6.0]
}
```

A function without an output can still be evaluated as a statement, which
runs it to completion.


> **Provisional: structural tuple elements.** For a structural
> `tuple<f64, f64>` parameter the agreed element shape is a tuple literal
> whose positions are values or `_` for a field that does not tick, so a
> bid-only cycle would be written `(1.0, _)`:
>
> ```hgl
> fn midpoint(tob: tuple<f64, f64>) -> f64 =>          # provisional structural replay
>     (tob[0] + tob[1]) / 2.0
>
> test midpoint_waits_for_both_sides {
>     assert eval(midpoint, tob: [(1.0, _), (_, 3.0), (2.0, _)])
>         == [_, 2.0, 2.5]
> }
> ```

## Timed sequences

> **Provisional.** Timed sequences and rolling-input replay use the agreed
> forms below; their remaining harness rules are separate design work.

When the timing matters, key each element by a time. A `duration` key is an
offset from the start of the run; a `datetime` key is an absolute instant and
fixes the run's start:

```hgl
use hgraph.std::{mean}

fn recent_mean(price: rolling<f64, 5m>) -> f64 => mean(price)   # provisional timed replay

test recent_mean_spans_five_minutes {
    assert eval(recent_mean, price: [0s: 1.0, 2m: 3.0, 5m: 5.0, 9m: 7.0])
        == [5m: 3.0, 9m: 5.0]
}
```

A timed expected sequence lists exactly the ticks the output produced, at
their times; there is no `_` because a silent instant is simply absent.
Within one `eval`, every sequence is either dense or timed.

## Running a module

A module has no `main`. Any `export fn` with no temporal parameters is an
*entry*; its `const` parameters are bound from the command line and its
output, if any, is the run's output:

```hgl
module examples.app

use hgraph.std::{schedule}

export fn heartbeat(const every: duration = 1s) -> datetime =>
    last_modified(schedule(every))
```

Run it from the command line; each tick of the entry's output is printed as
a `time value` line:

```text
hgl run examples/app.hgl --mode sim --start 2026-09-03T08:00:00Z \
    --end 3s --set every=1500ms
2026-09-03T08:00:01.5Z 2026-09-03 08:00:01.500000
```

A module with exactly one entry needs no `--entry`. `--set` values are HGL
constant expressions checked against the parameter's declared type
(`--set every=3` is `type: 'every' expects timedelta, got int`); a parameter
with a default may be left unset. `--start` takes a datetime with or without
its `@`, and `--end` a datetime or a duration after the start. The mode,
start, and end default to hgraph's own run defaults: a simulation starts at
the engine origin and ends when nothing remains scheduled; a real-time run
starts now.

> **Provisional: configuration file.** The agreed `--config run.toml` form
> mirrors the command line, with command-line options overriding the file.
>
> ```toml
> [run]
> entry = "heartbeat"
> mode = "realtime"
> start = 2026-09-03T08:00:00Z
> end = "1d"
>
> [run.params]
> every = "1500ms"
> ```
>
> TOML values would bind by their type: integer to `i64`, float to `f64`,
> string to `str` or, for a temporal parameter, the HGL literal spelling such
> as `"1d"`, offset date-time to `datetime`, local date to `date`, array to
> `list`.

The same module therefore runs as a backtest and as a live process with
nothing changed in the source. That is why the language has neither a
`main` nor a `run(fn, config)` form.

## The REPL

`hgl repl` accepts declarations and test bodies interactively. A `test`
block entered at the prompt runs immediately; a bare `eval(...)` at the
prompt prints the observed sequence, which is the quickest way to see what
a function does:

```text
hgl> let x = 1 + 2
hgl> x
3
hgl> fn midpoint(tob: atomic<tuple<f64, f64>>) -> f64 => (tob[0] + tob[1]) / 2.0
hgl> eval(midpoint, tob: [(1.0, 2.0), (2.0, 3.0)])
[1.5, 2.5]
hgl> test again { assert eval(midpoint, tob: [(2.0, 4.0)]) == [3.0] }
again ... ok
1 test, 0 failed
```

The REPL rebuilds the session after each accepted input, so what it shows is
what the same source does under `hgl test` and `hgl run`. A declaration
that does not check is reported and dropped; the session keeps its last
valid state. `let` and `var` bindings entered at the prompt persist; binding
a function to a name (`let half = midpoint`) is accepted but the name cannot
then be passed to `eval`, which requires an exact HGL function.
Input continues on the next line while a bracket is open; `:list` shows the
session, `:help` the commands, and `:quit` leaves.

On a terminal the prompt is an editable line: the arrow keys move and recall
earlier inputs, history persists across sessions in `~/.hgl_history` (or the
file named by `HGL_HISTORY`), and tab completes the `:` commands, the
declaration keywords, the kernel modules, and every name the session has
declared or bound. Piped input (`hgl repl < session.hgl`) reads plain lines,
so scripts behave as before; `HGL_NO_LINE_EDITING=1` forces that mode on a
terminal too.

The proposed [run-wide keyed-state foundation](../design/decisions/0016-eval-scalar-buffer-capabilities.md)
configures replay with ordinary const data and record with a const string
key. The reusable store binds const keys and exact value types before start;
hooks access those entries without name lookup or type dispatch. An entry
is not initialized merely by binding it. For the admitted scalar types,
[ordinary timed values](../../../library/ordinary_replay_record.md) supply
the replay list and recording: a present publication is an ordinary
`TimedValue<T>` with a datetime field and a `delta<T>` value field, which
reduces to the scalar T. Eval omits silent input
positions from that list and retains their dense horizon separately.
The test's eval arguments stay the same. The
[ordinary delta type extension](../design/ordinary-delta-types.md) supplies
the same `TimedValue<T>` spelling for the structural publication profile,
where T is the original shape and its value field holds the sparse delta; general
nullable ordinary sequences remain separate.

## First-pass limits

Compiler coverage and platform limits are implementation concerns, recorded
in the [audit notes](https://github.com/hhenson/hgraph_spec_audit/tree/main/docs/implementation-notes).

## What runs where

An implementation may interpret a graph or compile it to native code. Both
must preserve the same test outcomes, ticks and effects. Toolchain, cache and
platform requirements belong to the implementation's documentation.

The proposed [collection delta eval profile](../design/eval-collection-deltas.md)
uses `delta<T>(...)` literals for sets, fixed lists, bundles and integer-key maps,
including recursive child updates. The target still uses `delta_value(value)`;
eval owns replay and recording. Its ordinary nonempty publication admission
excludes empty events, invalidation and REF designation pending separate
contracts. The [finite atomic profile](../design/atomic-delta-publications.md)
uses complete ordinary values, including present empty lists, at admitted atomic
boundaries. `_` continues to mean no publication.
