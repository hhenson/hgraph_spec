# hgraph_spec

The public HGL language and runtime specification: rules, HGL examples,
reasoned traces and accepted decisions. No Python, C++ or Rust code lives here.

- [Default expression binding scope](language/docs/design/default-binding-scope.md)
- [Language model](language/docs/design/language-model.md)
- [Empty sparse delta application](language/docs/design/empty-delta-validity.md) (accepted)
- [Publication data and separately observed state](language/docs/design/publication-state-proposal.md) (proposed)
- [Execution-error assertions](language/docs/design/execution-error-assertions.md)
  and [source-rejection tests](language/docs/design/compile-rejection-fixtures.md)
  with their [error catalogue](language/docs/design/error-catalogue.md)
- [Source documentation](language/docs/design/documentation.md)
- [Language guide](language/docs/user-guide/language-tour.md)
- [Runtime model](runtime/overview.md) and [conformance method](runtime/conformance.md)
- [HGL compiler conformance](compiler/conformance.md)
- [Wiring](wiring/wiring.md): how every front end describes a graph
- [Library contracts](library/README.md): what standard library operators publish
- [Historical reference](historical/README.md)

[hgraph_spec_audit](https://github.com/hhenson/hgraph_spec_audit) owns executable
Python/C++ probes, experiments and measured results. Accepted expected traces
stay here; observations never silently replace them.
[hgraph_std](https://github.com/hhenson/hgraph_std) owns portable HGL library code.
[hgraph](https://github.com/hhenson/hgraph) owns the C++ compiler/runtime and
native providers. The Rust compiler/runtime implementation remains private.

Rules state observable behavior independently of a compiler or runtime.
Non-normative explanations may describe how an implementation could satisfy a
rule; they must not require particular source files, internal passes, native
symbols or layouts. Implementation status, source references, build recipes
and test locations belong in [audit notes](https://github.com/hhenson/hgraph_spec_audit/tree/main/docs/implementation-notes)
or the implementation repository. Agreed rules and open design questions stay
here; lack of implementation support does not change either. Sources and extraction revisions are recorded in
[PROVENANCE.json](PROVENANCE.json). This repository is a source/data package;
it has no importable Python package or native implementation.
