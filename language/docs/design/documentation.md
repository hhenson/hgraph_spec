# Source documentation

Status: agreed.

Use `/** ... */` before a declaration. Write a short summary, then Google-style
sections with reStructuredText content. Do not decorate lines with `*`.
Separate definitions with one blank line; keep documentation with its definition.

```hgl
/**
Compute the logistic sigmoid.

Args:
    x: Input value.

Returns:
    The sigmoid of ``x``.

Notes:
    Maps the input onto :math:`(0, 1)`:

    .. math::

        \sigma(x) = \frac{1}{1 + e^{-x}}
*/
native const fn sigmoid(x: f64) -> f64
```

## Sections

| Section | Meaning |
|---|---|
| `Args:` | Parameter descriptions keyed by declared name; omit repeated types. |
| `Type Args:` | Generic parameter descriptions keyed by declared name. |
| `Returns:` | Result meaning; the declaration supplies its type. |
| `Raises:` | Failures and their conditions. |
| `Ticks:` | Activation and publication behavior. |
| `Validity:` | Validity requirements and changes. |
| `Properties:` | Explanations keyed by declared domain and property. |
| `Requires:` | Explanations keyed by a declared requirement expression. |
| `Notes:`, `Examples:` | Further prose, code, math and diagrams. |

Sections start at column zero after removing the comment's common indentation.
Indent section contents four spaces; preserve additional indentation for reST.
An unknown heading remains prose. Structured keys must name declarations that
exist. Missing descriptions are allowed. Documentation never declares a type,
property, constraint, injectable or runtime behavior.

```hgl
/**
Add the current input values.

Args:
    lhs: Left input.
    rhs: Right input.

Type Args:
    L: Left value type.
    R: Right value type.
    O: Result value type.

Properties:
    <str, str, str>:
        associative:
            Regrouping preserves character order.
        identity:
            Leaves either operand unchanged.

    <i64, i64, i64>:
        commutative:
            Exchanging operands preserves the result.
        identity:
            Leaves either operand unchanged.

Notes:
    Integer associativity is not declared: regrouping can change
    which intermediate operation overflows.
*/
operator add_<L, R, O>(lhs: L, rhs: R) -> O
    properties<str, str, str> { associative, identity = "" }
    properties<i64, i64, i64> { commutative, identity = 0 }

/**
Implement addition through a native scalar operation.

Requires:
    native::add(L, R) -> O:
        An exact native value overload must exist. No implicit
        numeric widening is used.

Notes:
    .. mermaid::

        sequenceDiagram
            participant Node
            participant Native
            participant Output
            Node->>Native: add(lhs, rhs)
            Native-->>Node: Result
            Node->>Output: Publish result
*/
impl fn add_<L, R, O>(lhs: L, rhs: R) -> O
    requires native::add(L, R) -> O {
    when {
        return native::add(lhs, rhs)
    }
}
```

## Attachment and preservation

- A documentation comment attaches to the next module, function, native function,
  operator, struct, struct field or test declaration. Only whitespace may intervene.
- Ordinary comments remain ordinary comments. An unattached or repeated
  documentation comment is an error. Documentation inside executable bodies
  cannot document a later declaration.
- Remove delimiters, outer blank lines and common indentation. Preserve internal
  blank lines, Unicode, backslashes and relative indentation. Normalize line endings
  to LF. Indentation uses spaces or tabs; other Unicode whitespace is text.
  A same-line summary does not set the indentation of subsequent lines.
- Public behavior belongs on the interface declaration. An implementation or
  module part has its own documentation; selecting it never overwrites the interface.
- Preserve declaration identity, source association, signature, part and text through
  checking and lowering. Overloads and property domains remain distinct.
- Emit documentation with generated code and expose it to documentation tools.
  reST is the authoritative markup; passing it unchanged to a Markdown renderer
  does not constitute rendering support. Math uses `.. math::`; UML may use
  `.. mermaid::` sequence or class diagrams.
- Formatting must preserve attachment and content. Documentation-only changes do
  not change execution, overload resolution or a provider's semantic fingerprint.

## Acceptance scenarios

| Input | Expected result |
|---|---|
| Documented function, native declaration, module and field | Text attached to that declaration. |
| Math, Mermaid, Unicode and nested lists | Content and relative indentation survive lowering and emission. |
| CRLF and indented comments | Same normalized document as LF at top level. |
| Ordinary block comment or comment in a string | No documentation record. |
| Documentation followed by an ordinary comment or end of file | Unattached-document diagnostic. |
| Two documentation comments for one declaration | Diagnostic; never silently select one. |
| Unknown `Args` or `Type Args` key | Diagnostic at the documentation. |
| Unknown property, domain or requirement | Diagnostic; no new contract is created. |
| Two overloads with distinct docs | Separate records with their own declarations. |
| Shared native declaration and selected part | Both documents retained with separate ownership. |
| Formatter round trip | Same attachment, normalized text and contract. |
| Descriptor/documentation round trip | Same text, signatures and ownership. |
| Comment-only edit | Same checked behavior and semantic provider fingerprint. |

Compiler implementations test the scenarios for the language forms they support.
The smaller Rust compiler still rejects unsupported language forms explicitly.
