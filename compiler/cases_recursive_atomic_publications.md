# Recursive atomic publication cases

Check [finite recursive atomic values](../language/docs/design/recursive-atomic-publications.md)
through the existing generic delta pass-through.

| Case | Required result |
|---|---|
| Depth-three tree, equal repeated tree, silence, then leaf | Preserve complete trees and all publication slots; leaf replaces all prior descendants. |
| Self-recursive and same-module mutually recursive concrete declarations | Preserve exact nominal types at every edge; checking and shape matching terminate. |
| Finite generic recursive specialization admitted by ADR 0012 | Preserve its exact type arguments without expanding an unbounded type family. |
| Two trees differ only at a deep value or unset/present edge | Compare unequal; do not truncate comparison to root fields or shape. |
| Retain a tree with a mutable list/struct below a recursive edge | Source and one-capture mutation cannot alter sibling captures, stored values or another eval's recording. |
| Recursive value within an ordinary list or atomic child of a structural publication | Admit complete value transport; a recursive declaration edge through a container is still forbidden. |
| Empty/all-silent input with an exact atomic wrapper | Existing lifecycle and dense horizon; no inferred tree or fabricated leaf. |
| Required/non-atomic recursive edge, cyclic value, container recursion or expanding type arguments | Retain ADR 0012 rejection. |
| Abstract recursive family or a recursive nominal root used as a structural publication shape | Outside this concrete atomic admission. |
| Sparse optional-field clear, invalidation or references | Retain the separate unsupported boundaries. |
