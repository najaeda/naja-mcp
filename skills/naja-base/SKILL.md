---
name: naja-base
description: Generate Python scripts for Naja EDA transformations and manipulations. Use when creating scripts for netlist processing, design optimization, port manipulation, and circuit analysis. Provides reusable templates, common patterns, and comprehensive API reference for all 100+ NajaEDA functions and methods.
---

# Skill: Naja EDA Python Script Generation (Complete)

**Reference:** https://najaeda.readthedocs.io/en/latest  
**Complete API Catalog:** [resources/api-functions.md](resources/api-functions.md)  
**Patterns & Templates:** [resources/common-steps.md](resources/common-steps.md) | [resources/script-template.md](resources/script-template.md)  
**Rules:** [resources/api-rules.md](resources/api-rules.md)

---

## Mandatory Workflow (Read Before Coding)

Execute these steps **in order** before writing any code:

1. **Consult [api-functions.md](resources/api-functions.md)**
   - Verify exact function names, signatures, return types
   - Check for conditional APIs and requirements
   - Review error handling notes

2. **Check [common-steps.md](resources/common-steps.md)**
   - Find closest matching pattern for your use case
   - Adopt its structure and error checks

    - For Xilinx FPGA netlists, always load primitives first with `netlist.load_primitives('xilinx')`
    - Do not make primitive loading optional for Xilinx designs

3. **Copy from [script-template.md](resources/script-template.md)**
   - Use its import, logging, and argument-parsing structure
   - Follow error-handling patterns exactly

4. **If Not Documented**
   - Do NOT invent or guess
   - Ask for clarification instead

---

## NajaEDA Overview

**Framework:** Hierarchical netlist processor and optimizer for digital circuit design  
**Language:** Python 3  
**Module:** `import najaeda.netlist as netlist` (always use this)

### Key Concepts

- **Instance**: Hierarchical module or cell in design (top-level or sub-instance)
- **Net**: Signal wire (scalar or multi-bit bus) connecting terminals
- **Term**: Terminal/port of an instance (input, output, or inout)
- **Equipotential**: Logical signal connecting all terminals to same net
- **Leaf**: Primitive gate or lowest-level cell (no hierarchy)
- **Top**: Root instance of entire design hierarchy

---

## Complete Function Index by Category

### Part A: Loading (7 functions)
- `load_verilog(files, config)` → Instance
- `load_system_verilog(files, config)` → Instance
- `load_liberty(files)` → None
- `load_primitives(name)` → None
- `load_primitives_from_file(file)` → None
- `load_naja_if(path)` → None
- `reset()` → None

### Part B: Configuration Classes (3 classes)
- `VerilogConfig(keep_assigns, allow_unknown_designs, ...)`
- `SystemVerilogConfig(keep_assigns, elaborated_ast_json_path, ...)`
- `VerilogDumpConfig(dumpRTLInfosAsAttributes, dumpAssignsAsInstances)`

### Part C: Instance Methods (60+ methods)
Navigation, creation, queries, properties, modification, export

### Part D: Term Methods (35+ methods)
Connectivity, properties, access, modification

### Part E: Net Methods (25+ methods)
Properties, types, terminals, modification

### Part F: Equipotential Methods (8 methods)
Signal analysis, driver/reader queries

### Part G: Attribute Class (3 methods)
Metadata access

### Part H: Module Functions (20+ functions)
Navigation, statistics, transformation, export, utilities

### Part I: Instance Visitor Module
- `VisitorConfig(callback, args)`
- `visit(instance, config)` → Traverse hierarchy with callbacks

### Part J: Stats Module
- `dump_instance_stats_text(top, file)` → Design statistics

**→ Full details in [resources/api-functions.md](resources/api-functions.md) (Parts 1-13)**

---

## Quick Reference: 15 Most Common Operations

1. **Load design**
   ```python
   netlist.load_liberty([...])
   top = netlist.load_verilog("circuit.v", config=netlist.VerilogConfig(...))
   ```

2. **Get top module**
   ```python
   top = netlist.get_top()
   ```

3. **Navigate to instance**
   ```python
   inst = top.get_instance_by_path("top/cpu/alu")
   ```

4. **Get all child instances**
   ```python
   for child in top.get_child_instances(): ...
   leaves = list(top.get_leaf_children())
   ```

5. **Get input/output terminals**
   ```python
   inputs = list(inst.get_input_bit_terms())
   outputs = list(inst.get_output_bit_terms())
   ```

6. **Get terminals by name**
   ```python
   port = inst.get_term("signal_name")
   ```

7. **Get all nets**
   ```python
    named_nets = [n for n in inst.get_nets() if n.get_name()]
   nets = list(inst.get_nets())
   ```

8. **Connect/disconnect**
   ```python
    # Child instance term: use upper API to wire the parent-facing interface.
    term.connect_upper_net(net)
    term.disconnect_upper_net()

    # Top-level or lower-view manipulation: use lower API explicitly.
    term.connect_lower_net(net)
    term.disconnect_lower_net()
   ```

9. **Create nets and terminals**
   ```python
   net = inst.create_net("new_signal")
   term = inst.create_input_term("new_port")
   ```

10. **Check instance type**
    ```python
    if inst.is_leaf(): ... ; is_const0 = inst.is_const0()
    ```

11. **Get equipotential (fanout/drivers)**
    ```python
    equip = term.get_equipotential()
    drivers = equip.get_leaf_drivers()
    ```

12. **Apply optimization**
    ```python
    netlist.apply_dle()
    netlist.apply_constant_propagation()
    ```
    If a request explicitly forbids a native transform, compose the reusable
    truth-table, constant-source, and connectivity building blocks in
    [common-steps.md](resources/common-steps.md) instead of restoring that call.

13. **Export to Verilog**
    ```python
    top.dump_verilog("output.v")
    ```

14. **Visitor pattern (traverse)**
    ```python
    from najaeda import instance_visitor
    config = instance_visitor.VisitorConfig(callback=my_func, args=(data,))
    instance_visitor.visit(top, config)
    ```

15. **Get design stats**
    ```python
    max_fanout = netlist.get_max_fanout()
    max_level = netlist.get_max_logic_level()
    ```

---

## Essential Rules (Always Follow)

✗ **Never:**
- Invent API names not in [api-functions.md](resources/api-functions.md)
- Do not rely on standalone `connect()` or `disconnect()`; use the explicit upper/lower term APIs
- Import `naja_utils` (internal maintainer-only)
- Forget `list()` wrapping for generators
- Export without `.v` suffix or missing directory
- Silently fail without logging errors

✓ **Always:**
- For Xilinx FPGA netlists, call `netlist.load_primitives('xilinx')` before `load_liberty()` and `load_verilog()`
- If `load_primitives('xilinx')` fails, log an error and return 1 immediately
- Load Liberty **before** Verilog
- Check `top is None` after load
- Wrap generators: `list(inst.get_input_bit_terms())`
- Log errors with timestamps before returning non-zero
- Validate export path (suffix `.v`, directory exists)
- Use `netlist.apply_dle()` (native, not manual)
- Reference [api-rules.md](resources/api-rules.md) for forbidden APIs
- For a term that belongs to an instance child, prefer `connect_upper_net()` / `disconnect_upper_net()`; use lower only for the top or when the documentation explicitly asks for it
- Expect `RuntimeError` if you connect a term with the wrong design side; mismatch errors are usually a sign that upper/lower was inverted

---

## Common Pitfalls

1. Using `list(top.get_nets())[0]` as a candidate net. The first net is often anonymous, internal, or irrelevant.
2. Connecting a child term with `connect_lower_net()` when the operation should target the parent-facing interface.
3. Calling `disconnect()` alone. Use `term.disconnect_upper_net()` or `term.disconnect_lower_net()` explicitly.
4. Modifying a net with no observable fanout. Prefer a named net that reaches a primary output or a sequential element.
5. Assuming any net returned by `get_nets()` is useful. Filter named nets first before selecting a modification target.

---

## Mandatory Error Checks

Every script must validate:

```python
# 1. Input file exists
if not input_file.exists():
    logger.error(f"File not found: {input_file}")
    return 1

# 2. Load Liberty before Verilog
if is_xilinx_fpga_netlist:
    try:
        netlist.load_primitives('xilinx')
    except Exception as exc:
        logger.error(f"Failed to load Xilinx primitives: {exc}")
        return 1

netlist.load_liberty([...])

# 3. Check top loaded
top = netlist.load_verilog(...)
if top is None:
    logger.error("Failed to load design")
    return 1

# 4. Wrap all generators with list()
inputs = list(inst.get_input_bit_terms())

# 5. Validate export
if not output_path.suffix == ".v":
    logger.error("Output must end with .v")
    return 1
if not output_path.parent.exists():
    logger.error("Output directory missing")
    return 1

top.dump_verilog(str(output_path))
```

---

## Complete Script Template (Copy & Adapt)

```python
#!/usr/bin/env python3
import sys
import logging
from pathlib import Path

import najaeda.netlist as netlist
from najaeda import instance_visitor

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


def main() -> int:
    if len(sys.argv) < 2:
        print("Usage: script.py <input_file> [output_file]")
        return 1

    input_file = Path(sys.argv[1])
    output_file = Path(sys.argv[2]) if len(sys.argv) > 2 else Path("output.v")

    # Validate input
    if not input_file.exists():
        logger.error(f"Input not found: {input_file}")
        return 1

    # Validate output
    if output_file.suffix != ".v":
        logger.error(f"Output must end with .v: {output_file}")
        return 1
    if not output_file.parent.exists():
        logger.error(f"Output directory missing: {output_file.parent}")
        return 1

    # Load design
    try:
        design_dir = input_file.resolve().parent
        netlist.load_liberty([str(f) for f in design_dir.glob("*.lib")])
        top = netlist.load_verilog(
            str(input_file),
            config=netlist.VerilogConfig(
                keep_assigns=True,
                allow_unknown_designs=True
            ),
        )
        if top is None:
            logger.error("Failed to load design")
            return 1
    except Exception as e:
        logger.error(f"Load failed: {e}")
        return 1

    # Process (your code here)
    logger.info(f"Loaded: {top.get_model_name()}")
    leaves = list(top.get_leaf_children())
    logger.info(f"Leaves: {len(leaves)}")

    # Export
    try:
        top.dump_verilog(str(output_file))
        logger.info(f"Wrote: {output_file}")
    except Exception as e:
        logger.error(f"Export failed: {e}")
        return 1

    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

---

## Available Resources

| File | Purpose |
|------|---------|
| [api-functions.md](resources/api-functions.md) | Complete API reference: 100+ functions in 13 parts |
| [common-steps.md](resources/common-steps.md) | 5 approved workflow patterns |
| [script-template.md](resources/script-template.md) | Argument parsing & error handling templates |
| [api-rules.md](resources/api-rules.md) | Forbidden/deprecated API list (canonical) |
| [kepler-formal-mcp-tutorial.md](resources/kepler-formal-mcp-tutorial.md) | How to use Kepler Formal MCP for formality checks

**All scripts must reference these files, not invent API calls.**

## When To Use The Kepler Formal Tutorial

Open [resources/kepler-formal-mcp-tutorial.md](resources/kepler-formal-mcp-tutorial.md) when you need to:

- prepare a design for formal verification;

---

## Implementation Notes

- **Generator wrapping:** Methods like `get_input_bit_terms()` return generators — always wrap with `list()` before indexing or `len()`
- **Conditional APIs:** `load_primitives()` may not exist in all builds — use `hasattr(netlist, 'load_primitives')` check
- **Transform safety:** `apply_constant_propagation()` can segfault on some designs — use in separate process if critical
- **Export validation:** Parent directory must exist; suffix must be `.v`
- **Error logging:** Always log before returning non-zero exit code

---

**Last Updated:** May 2024  
**Maintained Alongside:** Comprehensive API catalog, pattern library, templates  
**For Full Details:** See [resources/api-functions.md](resources/api-functions.md) (Parts 1-15)
