# hgraph_spec

The public HGL language and runtime specification: rules, HGL examples,
reasoned traces and accepted decisions. No Python, C++ or Rust code lives here.

- [Language model](language/docs/design/language-model.md)
- [Source documentation](language/docs/design/documentation.md)
- [Language guide](language/docs/user-guide/language-tour.md)
- [Runtime model](runtime/overview.md) and [wiring](runtime/wiring.md)
- [Conformance method](runtime/conformance.md)
- [Historical reference](historical/README.md)

[hgraph_spec_audit](https://github.com/hhenson/hgraph_spec_audit) owns executable
Python/C++ probes, experiments and measured results. Accepted expected traces
stay here; observations never silently replace them.
[hgraph_std](https://github.com/hhenson/hgraph_std) owns portable HGL library code.
[hgraph](https://github.com/hhenson/hgraph) owns the C++ compiler/runtime and
native providers. The Rust compiler/runtime implementation remains private.

Implementation descriptions explain a rule; they do not make an ABI or layout
part of the language. Sources and extraction revisions are recorded in
[PROVENANCE.json](PROVENANCE.json). This repository is a source/data package;
it has no importable Python package or native implementation.
