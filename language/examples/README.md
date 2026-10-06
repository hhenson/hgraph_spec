# Language examples

These HGL files describe language behaviour. Compiler validation runs in
[hgraph's language tests](https://github.com/hhenson/hgraph/blob/main/language/tests/CMakeLists.txt)
after shared-source setup. That suite discovers examples, checks them, runs
supported `test` blocks and compiles generated fixtures. Native test support
varies by platform; this source-only repository has no CMake/CTest project.
The entries below identify the implementation tests for each example.
[Source-rejection examples](reject/README.md) use ordinary `hgl test`;
annotations isolate expected source errors alongside executable tests.

- [`eval-profile-errors.hgl`](eval-profile-errors.hgl) checks coded input-trace
  rejection before graph start, fresh-run membership and canonical positive
  controls. [`reject/delta-errors.hgl`](reject/delta-errors.hgl) isolates delta
  formation, exact-type and constructor source failures with surviving controls.
  These are normative fixtures; their presence is not an execution-validation claim.
- [`execution-errors.hgl`](execution-errors.hgl) checks the two catalogued
  yield failures, test continuation and nested expected-error assertions.
- [`contextual-local-bindings.hgl`](contextual-local-bindings.hgl) distinguishes
  ordinary node locals, graph scalars and graph connection rebinding.
- [`temporal-scalar-publications.hgl`](temporal-scalar-publications.hgl) forwards
  civil and named-zone scalar publications without changing their identities.
- [`atomic-delta-publications.hgl`](atomic-delta-publications.hgl) shows
  complete atomic replacement, defaults and present empty-list snapshots.
- [`ordinary-delta-types.hgl`](ordinary-delta-types.hgl) gives generic
  replay/record with typed structural delta storage, retained map updates,
  and empty ordinary data without empty-event application. See
  [ordinary delta types](../docs/design/ordinary-delta-types.md).
- [`generator-yield-operands.hgl`](generator-yield-operands.hgl) makes yield
  operand order, negative-duration failure and future resumption explicit,
  with distinct explicit and implicit checked target arithmetic examples.
  See the [yield operand rules](../docs/design/decisions/0015-pull-sources.md#operand-evaluation-resolution-and-retention).
- [`ordinary-replay-record.hgl`](ordinary-replay-record.hgl) gives generic
  scalar replay and record bodies using an ordinary `TimedValue<T>` struct
  whose value field has type `delta<T>`,
  generator traversal, typed global state and retaining list push. Its tests
  express repeated zero publications and empty/all-silent dense results under
  the [library data contract](../../library/ordinary_replay_record.md).
- [`struct-constructor-order.hgl`](struct-constructor-order.hgl) exposes
  ordinary constructor argument order through value-helper log messages,
  with successful field association and failure before a later argument.
  See the [constructor-order contract](../docs/design/struct-constructor-order.md).
- [`ordinary-list-values.hgl`](ordinary-list-values.hgl) specifies typed empty
  lists, length, indexed reads, end growth and independent nested retention
  under the [ordinary-list contract](../docs/design/ordinary-list-values.md).
  Its direct value-function assertions express expected language behavior.
- [`mutable-values.hgl`](mutable-values.hgl) specifies writable ordinary
  `var` contents, recursive read-only `let` access, and independent owning
  copies and construction. Its `test` uses value functions and does not
  invoke eval. It belongs to the proposed
  [value-mutability contract](../docs/design/value-mutability.md).
  This is a normative source example; compiler acceptance and execution of
  the extended content-mutation cases have not yet been validated.
- [`test-contexts.hgl`](test-contexts.hgl) keeps a runtime helper in a
  module-wide test scope shared by two contexts. Its two cases run through
  `hgraph_language_test_test-contexts`; cross-part visibility and production
  exclusion are covered by `hgraph_language_test_contexts` and emitter tests.
- [`midpoint.hgl`](midpoint.hgl) uses an internal helper, `export fn`, an
  atomic tuple, a `const` window, and a `test` of the unexported helper. It
  imports `hgraph.analytics`. Tests: 1 `test` under `hgl test`
  (`hgraph_language_test_midpoint`, gated on `hgraph::analytics`) and
  `generated_example_tests.cpp`.
- [`runtime-choice.hgl`](runtime-choice.hgl) contrasts a wiring-time topology
  choice, explicit time-series selection, and a `when` runtime function. It
  imports `hgraph.analytics`. Tests: none in the file;
  `generated_example_tests.cpp` builds it.
- [`pull-sources.hgl`](pull-sources.hgl) drives sources without temporal
  input: `inject alarm` for a one-shot wake-up, and `yield` generator sources
  with `while` loops, a bare `return`, an absolute time and a skipped past
  time ([ADR 0015](../docs/design/decisions/0015-pull-sources.md)). Tests:
  2 `test` blocks under `hgl test` (`hgraph_language_test_pull-sources`),
  discovered once hgraph's pin of this repository follows hgraph PR #1670.
- [`stateful-node.hgl`](stateful-node.hgl) demonstrates aggregate state,
  grouped injectables, lifecycle blocks, ordered handlers, previous output,
  and incremental collection output. Tests: none in the file; native behaviour
  in `generated_example_tests.cpp`, and the `--dump-hir` /
  `--dump-hgraph-ir` CTest cases read this example.
- [`when-defaults.hgl`](when-defaults.hgl) makes omitted and zero-argument
  handler selectors executable: `when {}` matches
  `when modified() && valid()`, while either omitted selector expands over all
  temporal parameters. Tests: 3 `test` blocks under `hgl test`
  (`hgraph_language_test_when-defaults`) plus focused generated-code checks.
- [`collection-views.hgl`](collection-views.hgl) demonstrates dual-phase
  `key_set`, runtime `keys`/`values`/`elements`/`items`, built-in and inline
  predicates,
  `last_modified`, and mutable lexical `var`. Tests: none in the file; native
  behaviour in `generated_example_tests.cpp`.
- [`const-debug.hgl`](const-debug.hgl) connects an HGL constant source to an
  HGL printing sink. Only integer printing is native; select the C++
  part under `impl/`. Tests: `generated_const_debug_tests.cpp` compiles the
  C++ binding and checks source ticks, duplicate sink ticks and fresh runs.
- [`lifecycle-capabilities.hgl`](lifecycle-capabilities.hgl) injects the
  node scheduler and the evaluation clock: a scheduler-driven source with no
  temporal input (`start { schedule(scheduler, 0s) }`, `when scheduled()`),
  `passivate(input)` after a count, and `clock.evaluation_time` (ADR 0010).
- [`native-provider.hgl`](native-provider.hgl) separates native scalar declarations
  from their providers and exercises injectable propagation through helpers.
  Implementations and provider binding tests live with each compiler/runtime.
- [`operators-and-generics.hgl`](operators-and-generics.hgl) demonstrates a
  nominal bodyless `operator`, a generic `impl fn` implementation, const-generic
  rolling-window sizes, an exported exact function, the default minimum window
  size, and a duration window. Operators, functions, and structs may carry
  the agreed `requires` constraint syntax. Tests: none in the file; generic
  resolution and window ticks in `generated_generic_tests.cpp` and, on the
  direct backend, `../tests/wiring/backend_coverage_tests.cpp`.
- [`structural-types.hgl`](structural-types.hgl) demonstrates a recursively
  temporal struct, `atomic<S>`, a type-generic struct, closed-set requirements,
  abstract-only inheritance with a default override, and a sparse delta in a
  runtime function, alongside temporal maps and an anonymous `fn`. Tests: none
  in the file; `generated_structural_tests.cpp`, and direct-wiring struct
  cases in `../tests/wiring/backend_tests.cpp`.
- [`recursive-fields.hgl`](recursive-fields.hgl) declares recursive struct
  fields (ADR 0012): a linked list and a generic tree whose edges are optional
  `atomic` fields, a construction whose edge is a port, and field access
  through an edge. Tests: 3 `test` blocks under `hgl test`
  (`hgraph_language_test_recursive-fields`), asserted again on the generated
  C++ in `generated_recursive_tests.cpp`.
- [`reference-routing.hgl`](reference-routing.hgl) demonstrates `ref<T>`
  parameters and results in runtime functions: forwarding a reference, and
  routing one element of a `list<ref<T>, 3>` by a temporal index. Tests: none
  in the file; routing ticks in `generated_reference_tests.cpp` and, for the
  composition shapes, `../tests/wiring/backend_coverage_tests.cpp`.
- [`fixed-list-iteration.hgl`](fixed-list-iteration.hgl) demonstrates a
  graph-phase `for` over a fixed temporal list with `elements` and `items`,
  wiring one body per child connection. Tests: none in the file; per-child
  wiring in `generated_iteration_tests.cpp` and
  `../tests/wiring/backend_tests.cpp`.
- [`dynamic-collection-iteration.hgl`](dynamic-collection-iteration.hgl)
  demonstrates a graph-phase `for` over a temporal map and an unbounded list,
  one sink child graph per key or index, with shared captures passed whole.
  Tests: none in the file; child-map ownership in
  `generated_iteration_tests.cpp` and recorded child ticks in
  `../tests/wiring/backend_coverage_tests.cpp`.
- [`conditional-result.hgl`](conditional-result.hgl),
  [`conditional-results.hgl`](conditional-results.hgl), and
  [`conditional-mixed-results.hgl`](conditional-mixed-results.hgl) exercise
  temporal branch results, escaping assignments, structural result packing,
  and remapping in both compiler backends. Tests: none in the files; ticks in
  `generated_tests.cpp` and `../tests/wiring/backend_tests.cpp`.
- [`conditional-forwarding.hgl`](conditional-forwarding.hgl) preserves an
  initialized result through an implicit or explicit unassigned branch and
  exercises independent reference forwarding for structural result fields.
  Tests: none in the file; ticks in `generated_tests.cpp` and
  `../tests/wiring/backend_tests.cpp`.
- [`conditional-omitted-else.hgl`](conditional-omitted-else.hgl) shows that a
  consumed temporal conditional without `else` produces no tick while false,
  using a type-resolved `nothing` branch in both compiler backends. Tests:
  1 `test` under `hgl test` (`hgraph_language_test_conditional-omitted-else`)
  and `generated_tests.cpp`.
- [`conditional-early-return.hgl`](conditional-early-return.hgl) returns from
  one temporal branch and composes the rest of the body as the other branch's
  continuation, for top-level, nested, tail, assigned, and outputless forms.
  Tests: 8 `test` blocks under `hgl test`
  (`hgraph_language_test_conditional-early-return`) and `generated_tests.cpp`.
- [`conditional-sinks.hgl`](conditional-sinks.hgl) controls child-graph
  lifetime with an outputless temporal conditional through the native sink
  switch, keeps a second sink outside it, and discards a sink conditional
  inside a value-producing graph. Tests: none in the file; `generated_tests.cpp`
  and `../tests/wiring/backend_tests.cpp`.

As compiler slices land, each example should advance from parsing and typed IR
coverage through `hgl test` to generated C++ behavior and backend parity.
The backend-parity module that is built both ways lives in
`../tests/codegen/parity.hgl`; the expression-embedded temporal conditional
is pinned there. The acceptance sequence is defined in the
[Developer Guide](https://github.com/hhenson/hgraph/blob/main/language/docs/developer-guide/testing-and-compatibility.md#documentation-examples).
