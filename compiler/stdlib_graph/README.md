# Standard-library constant to diagnostic sink

[The graph](../../language/examples/stdlib-const-debug.hgl) imports `const` and
`debug_print` from `hgraph.std`. It uses their HGL implementations, with only
scalar formatting and printing supplied by a target's native part.

`const(42)` arms `alarm` at start. Its plain `when` publishes once in the first
cycle. The sink prints `answer: 42` in that cycle and publishes nothing. There
is no second alarm. A fixed delay moves publication relative to the run's start.
Each call constructs its own node; a fresh execution starts a fresh tick counter.

The literal expectations in `cases.json` follow ADR 0015 and the library's
control-operator contracts. Tick cells are one microsecond apart, starting at
`MIN_START`. Missing cells do not admit the sink. Equal ordinary ticks count
separately. Sampling counts admitted ticks, so sample 2 prints only the second
42 in `[_, 42, _, 42, -7]`. Backend timestamp prefixes are excluded from the
comparison; the label, sampling prefix and scalar text are retained.

The earlier `const_`/`print_i64` bootstrap remains a compiler regression fixture.
It does not exercise the current imported standard-library operators.
