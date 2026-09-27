# Declaration formatting

Formatting exposes declaration ownership without changing language semantics.

- Separate definitions by one blank line. Keep consecutive imports together.
- A signature, its `requires` clause and its `properties` clauses form one
  definition. Put each clause on a new line, indented four spaces relative to
  the declaration. No blank line separates a clause from its owner.
- Shift multiline clause continuations with the clause. Preserve their relative
  indentation and preserve literal and comment contents.
- Keep same-line comments with the preceding definition. Keep standalone
  comments before the following definition; place its separating blank line
  before those comments.
- Apply definition spacing inside `test { ... }` contexts too.

```hgl
operator add_<L, R, O>(lhs: L, rhs: R) -> O
    properties<str, str, str> { associative, identity = "" }
    properties<i64, i64, i64> { commutative, identity = 0 }

operator sub_<L, R, O>(lhs: L, rhs: R) -> O
```

The first formatter covers declaration layout. Expression spacing, body
indentation and line wrapping retain their existing spelling. It preserves
LF or CRLF line-ending style and supplies a missing final newline. Invalid syntax is
reported without rewriting the file; unresolved imports do not prevent
formatting. Formatting twice produces the same result, with the same parsed
structure and unchanged comments and literals.

Expected checks: adjacent operators gain a blank line; attached clauses stay
with their operator; comment preambles remain attached; imports stay grouped;
nested test definitions separate; CRLF stays CRLF; malformed input is unchanged.
