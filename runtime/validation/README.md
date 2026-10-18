# Expected runtime observations

These files contain required inputs, snapshots, traces and accepted decisions;
they do not record implementation results.

Fixed-collection and description snapshots include the effective corrections
and accepted decision assertions. Historical intermediate expectations and
comparison notes are preserved in the audit archive. Decision assertions name
the same expected values directly; consumers do not reapply historical overlays.

The parity case set excludes exact container string escaping: quoting and
escaping remain open, and `repr` is outside the standard. Other case results
are unchanged. Measured runs remain in
[hgraph_spec_audit](https://github.com/hhenson/hgraph_spec_audit).
