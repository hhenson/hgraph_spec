# Architecture decision records

Numbered records preserve language and runtime decisions, including explicit
open questions. Compiler architecture, serialization recipes and implementation
progress belong to the implementation or audit repository.

- [0005: Module-local exact native functions may contain C++](0005-inline-cpp-native-functions.md)
- [0006: Explicit source parts form one logical module](0006-multi-file-module-parts.md)
- [0007: Explicit parameter-pack shapes](0007-parameter-packs.md)
- [0008: Temporal programming, value functions, and target mappings](0008-temporal-contracts-and-target-mappings.md)
- [0009: Native functions may raise, under hgraph's node error model](0009-native-errors-and-the-node-error-model.md)
- [0010: Clock and scheduler capabilities, scheduled handlers, and input activity](0010-lifecycle-capabilities.md)
- [0011: `cache` declarations](0011-cache-declarations.md)
- [0012: Recursive struct fields](0012-recursive-struct-fields.md)
- [0013: Struct imports](0013-struct-imports.md)
- [0014: Native implementation interfaces](0014-native-implementation-interfaces.md)
- [0015: Pull sources: the `alarm` injectable, `yield`, and `while`](0015-pull-sources.md)

- [0016: Typed scalar buffer capabilities for replay and record](0016-eval-scalar-buffer-capabilities.md) (proposed)
