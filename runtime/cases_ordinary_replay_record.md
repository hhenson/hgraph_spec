# Ordinary scalar replay and recording cases

These expected observations exercise the
[ordinary scalar data contract](../library/ordinary_replay_record.md).
Harness `_` denotes no publication; it is not an ordinary list element.
Let s be the run start and d the minimum engine step. Unless stated otherwise,
the run end is later than every supplied time and there are no other writers
to the recorder's entry.

## TIMED-SCALAR — payload and silence

For each row, supply entries at s+2d, s+4d, s+5d and s+8d with payloads
A, A, B, A. Record an explicit scalar delta pass-through. The recording has
exactly those four timestamp/payload pairs, in that order. With dense input
horizon 11 the result is `[_, _, A, _, A, B, _, _, A, _, _]`.

| T | A | B |
|---|---|---|
| `bool` | `false` | `true` |
| `i64` | `0` | `-7` |
| `f64` | `0.0` | `1.5` |
| `str` | `""` | `"delta"` |
| `date` | `@1970-01-01` | `@2024-02-29` |
| `time` | midnight | 12:34:56 |
| `datetime` | `@1970-01-01T00:00:00Z` | `@2024-02-29T12:34:56Z` |
| `duration` | `0s` | `1s` |

The second A is a publication even though equal to the first. False, zero,
empty text and midnight are never treated as absent.

## TIMED-EMPTY — input data and dense horizon differ

For each scalar T, an empty `list<TimedValue<T>>` publishes nothing. A
started recorder has an initialized empty list. For a typed pass-through,
eval with four silent input slots yields four silent result slots; eval
with zero input slots yields an empty result. The ordinary replay data and
recording are empty in both runs. Neither invents absent stored elements.

## TIMED-ORDER — existing generator and engine boundaries

| Timed entries encountered in list order | Expected publications |
|---|---|
| `(s, 0)`, `(s+2d, 0)` | Both, at their supplied times. |
| `(s-d, 1)`, `(s, 2)` | Only 2 at s; past entry skipped. |
| `(s+2d, 1)`, `(s+d, 2)` | Ordering error at the second entry; no successful trace. |
| `(s-2d, 1)`, `(s-d, 2)`, `(s, 3)` | Only 3 at s; increasing past entries skipped. |
| `(s-d, 1)`, `(s-d, 2)` | Ordering error at the second entry, despite both times being past. |
| `(s-d, 1)`, `(s-2d, 2)` | Ordering error at the second entry, despite both times being past. |
| `(s, 1)`, `(s, 2)` | Ordering error at the second entry; no sorting, merging or successful trace. |
| One entry exactly at exclusive run end | No publication. |
| One entry after run end | No publication. |

No row claims pre-start ordering validation. Earlier successful effects of
a failed run follow the node error contract; failure is not a successful
dense result. Engine sentinel/range rules apply to scheduled and published
times; a past absolute entry is skipped only after passing ordering validation.
The past-pair cases also apply when both timestamps precede the engine epoch.

## TIMED-TYPES — ordinary exact types

An empty list with expected type `list<TimedValue<i64>>` fixes replay's
result to i64. An untyped empty list fails ordinary checking. A datetime
field with a str value, an i64 payload field with a bool value, or a missing
required field fails ordinary struct construction checking. A fixed-size list
does not silently convert to replay's unbounded list type. A list containing
`_` or `null` is not this ordinary data representation.

Record of an i64 input binds its key to `list<TimedValue<i64>>`; a different
exact type requirement at the same key fails under the existing prepared-entry
rules before start. Structural T, `signal`, references and windows are not
admitted by these scalar operator requirements.

## TIMED-RETAIN — lifecycle and failure

Immediately after a recorder starts, its typed entry is present and has
length zero. Publish A then B: the entry has length two and its first entry
still owns A with the first evaluation timestamp. Later input changes cannot
change it. Teardown without more publications adds nothing. After all stop
hooks, the run owner obtains the same two entries as an independent result.

An absent entry during required extraction fails; it is not equivalent to
the initialized length-zero recording. A failed capture construction or push
does not add a partial capture or change prior captures; earlier completed
effects remain and the enclosing run reports failure. A separate fresh run
starts with its own empty recording. This case assumes no competing writer
and adds no same-key collision or checkpoint/restart policy.
