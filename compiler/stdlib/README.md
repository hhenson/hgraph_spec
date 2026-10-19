# Standard-library behavior cases

These cases specify expected ticks without selecting a public library namespace.
Each runner supplies the following case entry points:

| Entry point | Behavior |
| --- | --- |
| `bit_and_i64(lhs, rhs)` | Bitwise AND of the latest valid integer inputs; either input activates. |
| `sample_i64(signal, ts)` | Publish the latest valid `ts` only when `signal` ticks. |
| `dedup_i64(ts)` | Publish the first valid value and subsequent changes. |

[cases.json](cases.json) gives inputs, expected ticks and reasoning;
[cases.hgl](cases.hgl) expresses the same expectations as HGL tests.
