Runtime behaviour specification
===============================

The concept-first runtime model describes intended behaviour, independent of
storage and implementation language. Its chapters, conformance cases and
source evidence distinguish established behaviour, proposals and open work.
It is incomplete and does not certify an implementation.

Wiring (how a graph is described, and how calls resolve) is specified apart
from the runtime in ``wiring/``; library operator contracts in ``library/``.
The runtime begins where wiring ends: at the builder boundary.

.. toctree::
   :maxdepth: 1

   overview
   execution_engine
   graph
   node
   time_series
   scalar_types
   injectables
   conformance
   open_points
   design_options
   cases_atomic
   cases_collections
   cases_windows
   cases_growing_lists
   cases_lifecycle
   cases_references
   cases_nested
   cases_fixed
   cases_sources
   cases_engine
   cases_injectables
   cases_scalar
   wiring <../wiring/wiring.md>
   wiring cases <../wiring/cases_wiring.md>
   library operator contracts <../library/operator_contracts.md>
   validation <https://github.com/hhenson/hgraph_spec_audit/blob/main/runtime/validation.md>
   validation/README <https://github.com/hhenson/hgraph_spec_audit/blob/main/runtime/validation/README.md>
   validation/fixed/README <https://github.com/hhenson/hgraph_spec_audit/blob/main/runtime/validation/fixed/README.md>
   validation/parity/README <https://github.com/hhenson/hgraph_spec_audit/blob/main/runtime/validation/parity/README.md>
   validation/wiring/README <https://github.com/hhenson/hgraph_spec_audit/blob/main/runtime/validation/wiring/README.md>
   representations
   layout_example
   boundaries
   evidence <https://github.com/hhenson/hgraph_spec_audit/blob/main/runtime/evidence.md>
   extraction

The `Python-era specification <https://github.com/hhenson/hgraph_spec_audit/tree/main/historical/python>`_ retains the earlier Python-era documents for
historical context and domains outside these runtime chapters. HGL source
syntax remains in the repository's ``language/docs/`` documentation.
