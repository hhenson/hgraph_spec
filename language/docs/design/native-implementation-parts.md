# Native implementation parts

The shared declaration owns the callable signature. A selected implementation
part supplies execution shape, injectables and lifecycle requirements.
Shared HGL contracts cross repositories; implementations belong to their runtime.
A library selects one implementation, never both.

```hgl
module std.native
native fn filter(value: i64, const limit: i64) -> i64
```

A graph implementation has a body without `when`:

```hgl
module std.native part cpp_impl
native fn filter(value: i64, const limit: i64) -> i64 {}
```

A node implementation declares `when;` and optional lifecycle hooks:

```hgl
module std.native part cpp_impl
native fn filter(value: i64, const limit: i64) -> i64 {
    inject out, logger
    start;
    when;
    stop;
}
```

- No body means declaration only; `{}` is not a missing implementation.
- `native const fn` remains a value helper, including with an empty body.
- An implementation completes exactly one matching declaration. Match names,
  parameter names/types/constness, generics and complete canonical result type,
  including any `result<T,error>` recovery contract.
  Duplicate declarations or selected implementations are errors.
- Select parts explicitly in the target build. Part names and folder names
  have no target-selection magic. One unnamed interface file may accompany
  named parts. Assembly order does not affect matching.
- Injections belong to the selected implementation, not the shared signature.
  Value-call requirements silently propagate to callers, transitively and
  without duplicates. A constructed node owns its output and scheduler;
  its graph-construction caller does not borrow those node capabilities.
- Target requests may differ. Observable behaviour must still satisfy the
  same function specification. Missing services are target-checking errors.
- `start;` and `stop;` require `when;`. Hooks are unique and are not bodies.
  A value implementation cannot declare lifecycle hooks, `out` or `scheduler`.
- Node lifecycle follows engine activation and teardown, including nested
  scopes. Native construction and destruction do not replace these hooks.
- A call inside `when` creates no implicit child node. Explicit borrowed-TS
  helper syntax and temporal provider execution remain separate ABI work.

## Acceptance cases

These cases specify binding behavior. Helpers and clock observations must
preserve their independently specified runtime traces.

| Source/selection | Expected result |
| --- | --- |
| Bare declaration | Signature checks; emission requires a selected implementation |
| Matching `{}` | One temporal graph implementation |
| Matching `{ when; }` | One temporal node implementation |
| Matching `{ start; when; stop; }` | Node with start and stop binding requirements |
| Matching value implementation with logger | One value callable; callers silently acquire logger |
| Select a logger-requiring part or a part without that request | Same signature; selected requirements differ |
| Reverse part order | Same selected callable and requirements |
| Changed type, constness or complete canonical result type | No matching declaration; reject |
| Two implementations or two identical declarations | Reject, never another overload |
| Hook repeated, value hook, start/stop without when | Reject |
| Native temporal call inside node evaluation | Reject; no implicit lifecycle or ownership |
| Export/import selected metadata | Preserve shape, hooks and capabilities |

## Binding boundary

A temporal graph or node requires a provider binding that preserves its
execution shape and lifecycle. An implementation lacking that binding must
diagnose the contract rather than emit a scalar value call.
