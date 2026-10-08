# Unset required-read cases

These cases follow [required reads of unset ordinary observations](../language/docs/design/unset-required-reads.md).
Use an independently partial source with a valid sibling, retained before reading.

| Case | Required observation |
| --- | --- |
| Unset scalar arithmetic or Boolean condition | Raise `value.unset_read`; no result or chosen branch. |
| Unset fixed list length | Raise the same code, despite its known declared size. |
| Unset map iteration | Raise the same code before entering the loop body. |
| Present zero, false, list and map | Read their ordinary payloads successfully. |
| Retain a partial tuple and project/retain its unset child | Preserve absence without requiring the payload. |

The [HGL tests](../language/examples/unset-required-reads.hgl) catch each failure
and run present controls. Bounds errors, missing map keys, wholly invalid
source admission and empty-event publication are outside these cases.
