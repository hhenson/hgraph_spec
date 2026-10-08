# Unset required-read cases

These cases follow [required reads of unset ordinary observations](../language/docs/design/unset-required-reads.md).
Use an independently partial source with a valid sibling, retained before reading.

| Case | Required observation |
| --- | --- |
| Unset scalar arithmetic or Boolean condition | Raise `value.unset_read`; no result or chosen branch. |
| Unset fixed list length | Raise the same code, despite its known declared size. |
| Present zero, false, true and fixed list | Read their ordinary payloads successfully. |
| Retain a partial tuple and project/retain its unset child | Preserve absence without requiring the payload. |

The [HGL tests](../language/examples/unset-required-reads.hgl) catch each failure
and run present controls. Bounds errors, missing map keys, wholly invalid
source admission and empty-event publication are outside these cases.

Projected-child `let` retention is part of this extension. These fixtures state
required source behavior; their presence does not claim implementation support.
