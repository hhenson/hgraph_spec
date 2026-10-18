# Const and debug compiler bootstrap

The first complete compiler case connects two HGL nodes. `const_` has a fixed
`i64` argument, schedules itself at start, and publishes once in `when scheduled()`.
`debug_print` consumes a valid tick and calls the native value helper `print_i64`.
The graph connects the local helper `const_(42)` to `debug_print`. This
bootstrap helper is separate from the standard-library `const` operator.

Only integer formatting and console output are native. Source scheduling,
admission, publication and composition stay in HGL. Target parts complete the
same native declaration; the helper returns no value and creates no node.
This is an i64 bootstrap, not the full generic, labelled standard-library API.

## Expected observations

| Case | Expected |
| --- | --- |
| Start graph | One value tick, 42, at the starting evaluation time; print `42` once |
| Continue without input or alarm | No further tick or print |
| Fresh second execution | Print `42` once again; no shared node state |
| Source configured with -7 | One tick carrying -7 |
| Sink receives absent, 42, absent, 42, -7 | Print 42, 42, -7; absent cells do nothing |
| Construct/check/emit without running | No printing |
| Source with bare `when` and an armed schedule | Publishes when scheduled; no input subscription is required |
| Missing implementation or wrong native signature | Reject before native execution |
| Temporal input passed as fixed source configuration | Type error |

Compare values, ticks and effects against these expectations. The bootstrap
prints only the integer and a newline.
