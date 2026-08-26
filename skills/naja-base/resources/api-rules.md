# Canonical: Forbidden and Deprecated API Rules

This file centralizes forbidden, deprecated, or otherwise disallowed Naja API calls and export patterns.

Maintainers: update this single file; other documentation references it instead of repeating the list.

## Forbidden / Non-existent APIs (Do NOT use)
- `netlist.export_verilog()`
- `netlist.export_netlist()`
- `naja_lib.load_verilog()`
- `naja_lib.apply_del()`
- `VerilogDumpConfig.dump()`
- `term.disconnect()`

## Export guidance
- Preferred export: call `top.dump_verilog("output.v")` on the top instance.
- Output path must end with `.v` and the parent directory must exist prior to export.

## Loading / Compatibility Rules
- Use `netlist.load_primitives('xilinx')` only for Xilinx FPGA netlists. Call it before `load_liberty()` and `load_verilog()`, and abort on failure.
- For standard ASIC or Liberty-based flows, load `.lib` files and do not call `load_primitives('xilinx')` unless the workflow explicitly requires it.

## Hierarchy / Connectivity Rules
- Do not call `disconnect()` by itself for term rewiring. Use `term.disconnect_upper_net()` or `term.disconnect_lower_net()` explicitly.
- For a leaf child created with `create_child_instance()`, always use `connect_upper_net()` / `disconnect_upper_net()` on its terms when wiring from the parent side.
- Use `connect_lower_net()` / `disconnect_lower_net()` only for top-level terms or when the documentation explicitly asks for the lower side.
- A wrong upper/lower choice can trigger `RuntimeError` mismatch failures because the term is being attached to the wrong design side.

## API Ownership

- `Instance` owns hierarchy and design contents: `get_leaf_children()`,
  `get_input_bit_terms()`, `get_output_bit_terms()`, `get_net()`, `create_net()`,
  and `create_child_instance()`.
- `Term` owns port behavior and connectivity: `get_truth_table()`,
  `get_equipotential()`, `get_upper_net()`, `get_lower_net()`, and the explicit
  upper/lower connect and disconnect methods.
- `Equipotential` owns signal-wide analysis: constant predicates and leaf/top
  driver and reader queries.
- `Net` owns net properties and connected-term queries: `set_type()`,
  `get_terms()`, `get_inst_terms()`, and `get_design_terms()`. A `Net` does not
  have input or output bit-term generators.
- A `get_*()` lookup can return `None`. When the request says to create an
  object, call the matching `create_*()` method rather than assuming a named
  object already exists.
- Connect a created driver's output `Term` to its parent `Net`; do not attempt to
  connect an `Instance` or `Net` as though it were a terminal.

## Notes for script generators
- Do not copy this list into other files. Reference this canonical file when producing documentation or validation logic.
