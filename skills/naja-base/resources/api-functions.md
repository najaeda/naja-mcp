# Catalog: Complete Naja EDA API Functions

**Reference:** https://najaeda.readthedocs.io/en/latest
**Module:** `import najaeda.netlist as netlist`
**Version:** Complete API as of 2024

This is the **comprehensive** catalog of all documented Naja functions, classes, and methods. Always consult this before generating a script.
If a function is not listed here, do not use it — ask for clarification instead.

---

## Pièges de performance

### TOUJOURS utiliser set() pour les collections visitées
```python
# CORRECT — O(1)
visited = set()
visited.add(term)
if term in visited: ...

# INCORRECT — O(n), catastrophique sur grands designs
visited = []
visited.append(term)
if term in visited: ...
```

### Ordre BFS correct — dédoublonner AVANT d'ajouter dans la queue
```python
# CORRECT
if term not in visited:
  visited.add(term)
  queue.append(term)

# INCORRECT — doublons dans la queue
queue.append(term)
if term in visited:
  continue
visited.add(term)
```

### Types NON hashables — ne jamais mettre dans un set() directement
- Equipotential -> utiliser term.key() du Term qui y mène
- Net -> utiliser net.key()
- (Term et Instance sont hashables directement)

---

## Part 1: Loading Functions (Module Level)

### load_verilog(files, config=None) → Instance
**Objective:** Load Verilog design(s) from file(s)
- **Input:** File path (string) or list of file paths; optional VerilogConfig
- **Output:** Design object loaded in memory
- **Return:** Instance (top-level module)
- **Example:**
```python
top = netlist.load_verilog(
    "circuit.v",
    config=netlist.VerilogConfig(keep_assigns=True, allow_unknown_designs=True)
)
```
- **Raises:** Exception if no files provided
- **Notes:** Most common format for digital designs

### load_system_verilog(files, config=None) → Instance
**Objective:** Load SystemVerilog design(s) from file(s)
- **Input:** File path (string) or list of file paths; optional SystemVerilogConfig
- **Output:** Design object loaded in memory
- **Return:** Instance (top-level module)
- **Example:**
```python
top = netlist.load_system_verilog(
    "design.sv",
    config=netlist.SystemVerilogConfig(keep_assigns=True)
)
```
- **Raises:** Exception if no files provided

### load_liberty(files) → None
**Objective:** Load Liberty library file(s)
- **Input:** File path (string) or list of file paths
- **Output:** Library object with cell definitions loaded in memory
- **Return:** None
- **Example:**
```python
netlist.load_liberty([str(f) for f in Path(design_dir).glob("*.lib")])
```
- **Raises:** Exception if no files provided
- **Notes:** 
  - Always load `.lib` files **before** Verilog
  - Defines timing and gate characteristics
  - For Xilinx FPGA netlists, load primitives first with `netlist.load_primitives('xilinx')`

### load_primitives(name) → None
**Objective:** Load embedded primitive library
- **Input:** Library name (string)
- **Output:** Primitives object loaded
- **Return:** None
- **Example:**
```python
netlist.load_primitives('xilinx')  # or 'yosys'
```
- **Supported:** 'xilinx', 'yosys'
- **Raises:** ValueError if name not recognized
- **Notes:**
  - For Xilinx FPGA netlists, call `netlist.load_primitives('xilinx')` first, before `load_liberty()` and `load_verilog()`
  - If `load_primitives('xilinx')` fails, treat it as fatal and stop immediately
  - Use when Liberty files unavailable for non-Xilinx flows

### load_primitives_from_file(file) → None
**Objective:** Load primitives library from Python file
- **Input:** File path (string)
- **Output:** Primitives library loaded from code
- **Return:** None
- **Example:**
```python
netlist.load_primitives_from_file("primitives.py")
```
- **Notes:** 
  - The file must define a `load(db)` function that will be called to populate the database
  - Useful for custom primitive libraries beyond xilinx/yosys

### load_naja_if(path) → None
**Objective:** Load a Naja IF file
- **Input:** Path to Naja IF file (string)
- **Output:** Current netlist restored from disk
- **Return:** None
- **Example:**
```python
netlist.load_naja_if("design.naja")
```
- **Notes:** Use when round-tripping a design in Naja IF format

---

## Part 2: Configuration Classes

### VerilogConfig (class)
Configuration object for Verilog loading
```python
config = netlist.VerilogConfig(
    keep_assigns=True,                    # Keep assign statements as instances
    allow_unknown_designs=False,          # Allow designs not in library
    preprocess_enabled=False,             # Enable preprocessing
    conflicting_design_name_policy='forbid'  # Policy for name conflicts
)
top = netlist.load_verilog("design.v", config=config)
```

**Public Attributes:**
- `keep_assigns: bool = True`
- `allow_unknown_designs: bool = False`
- `preprocess_enabled: bool = False`
- `conflicting_design_name_policy: str = 'forbid'` - Policies: 'forbid', 'allow', 'rename'

### SystemVerilogConfig (class)
Configuration object for SystemVerilog loading
```python
config = netlist.SystemVerilogConfig(
    keep_assigns=True,                    # Keep assign statements as instances
    elaborated_ast_json_path=None,        # Path to elaborated AST JSON
    diagnostics_report_path=None,         # Path to diagnostics report
    pretty_print_elaborated_ast_json=True,
    include_source_info_in_elaborated_ast_json=True,
    flist=None,                           # File list path
    top=None,                             # Top module name
    suppress_warnings=None                # List of warnings to suppress
)
top = netlist.load_system_verilog("design.sv", config=config)
```

**Public Attributes:**
- `keep_assigns: bool = True`
- `elaborated_ast_json_path: str = None` - Optional path to save elaborated AST as JSON
- `diagnostics_report_path: str = None` - Optional path to save diagnostics report
- `pretty_print_elaborated_ast_json: bool = True` - Pretty-print AST JSON output
- `include_source_info_in_elaborated_ast_json: bool = True` - Include source location in AST
- `flist: str = None` - File list (.f file) path for includes and defines
- `top: str = None` - Top module name (if not first module)
- `suppress_warnings: list = None` - List of warning patterns to suppress

### VerilogDumpConfig (class)
Configuration object for Verilog export
```python
config = netlist.VerilogDumpConfig(
    dumpRTLInfosAsAttributes=False,    # Dump RTL info as attributes
    dumpAssignsAsInstances=False       # Dump assigns as instances
)
top.dump_verilog("output.v", config=config)
```

**Public Attributes:**
- `dumpRTLInfosAsAttributes: bool = False` - Export RTL metadata as Verilog attributes
- `dumpAssignsAsInstances: bool = False` - Export assign statements as instance instantiations

---

## Part 3: Instance Class (Methods)

### Instance Overview

In najaeda, an `najaeda.netlist.Instance` encapsulates the concept of an instance in its hierarchical context.

When an Instance is modified through editing methods, najaeda will automatically manage the necessary uniquification.

### Generic Gates (and, or, etc.)

najaeda supports generic gates as defined in Verilog:

- **n-input gates:** and, nand, or, nor, xor, xnor
  - Single scalar output + bus input terminal (size n)
  - All unnamed terminals

- **n-output gates:** buf, not
  - Scalar input + bus output terminal (size n)
  - All unnamed terminals

### Instance Attributes

All Instance methods return typed objects (Instance, Term, Net, Attribute) that can be further manipulated.

---

### Navigation & Inspection
```python
top = netlist.get_top()
inst = top.get_instance_by_path("top/cpu/alu")  # hierarchical path
child_inst = top.get_child_instance("cpu")      # by name or list
child_inst = top.get_child_instance_by_id([1, 2])  # by IDs

# Iteration
for child in top.get_child_instances(): pass     # all children one level
for leaf in top.get_leaf_children(): pass        # all leaves (primitives)

# Get references
design = inst.get_design()  # parent instance
model_id = inst.get_model_id()  # returns tuple(int, int, int)
model_name = inst.get_model_name()  # model name string
name = inst.get_name()  # instance name
```

### Term/Net Queries
```python
inputs = list(inst.get_input_bit_terms())      # all scalar input terms
outputs = list(inst.get_output_bit_terms())    # all scalar output terms
input_terms = list(inst.get_input_terms())     # all input terms (bus+scalar)
output_terms = list(inst.get_output_terms())   # all output terms
bit_terms = list(inst.get_bit_terms())         # all bit-level terms
all_terms = list(inst.get_terms())             # all terms

nets = list(inst.get_nets())                   # all nets (bus+scalar)
bit_nets = list(inst.get_bit_nets())           # all bit-level nets
net = inst.get_net("signal_name")              # get net by name

term = inst.get_term("port_name")              # get term by name
```

### Creation Methods (Scalar)
```python
net = inst.create_net("new_signal")            # scalar net
input_term = inst.create_input_term("in_port")
output_term = inst.create_output_term("out_port")
inout_term = inst.create_inout_term("inout_port")
```

### Creation Methods (Bus)
```python
bus_net = inst.create_bus_net("addr", msb=31, lsb=0)  # 32-bit bus
bus_input = inst.create_input_bus_term("data", msb=7, lsb=0)
bus_output = inst.create_output_bus_term("result", msb=15, lsb=0)
bus_inout = inst.create_inout_bus_term("gpio", msb=3, lsb=0)

# Generic: specify direction
term = inst.create_term("port", direction=Term.Direction.INPUT)
term = inst.create_bus_term("addr", msb=31, lsb=0, direction=Term.Direction.INPUT)
```

### Child Instance Creation
```python
child = inst.create_child_instance(
    model="AND2",      # model name
    name="gate1"       # instance name
)
```

### Counting Methods
```python
n_attrs = inst.count_attributes()         # number of attributes
n_terms = inst.count_terms()              # total terms (scalar+bus)
n_bit_terms = inst.count_bit_terms()      # bit-level term count
n_input_terms = inst.count_input_terms()
n_input_bit_terms = inst.count_input_bit_terms()
n_output_terms = inst.count_output_terms()
n_output_bit_terms = inst.count_output_bit_terms()
n_nets = inst.count_nets()                # total nets
n_bit_nets = inst.count_bit_nets()        # bit-level net count
n_children = inst.count_child_instances()
```

### Property Tests
```python
is_leaf = inst.is_leaf()              # True if primitive
is_primitive = inst.is_primitive()    # True if built-in gate
is_top = inst.is_top()                # True if top design
is_blackbox = inst.is_blackbox()      # True if blackbox
is_sequential = inst.is_sequential()  # True if sequential cell

# Gate type checks
is_buf = inst.is_buf()        # True if buffer
is_inv = inst.is_inv()        # True if inverter
is_const = inst.is_const()    # True if constant generator
is_const0 = inst.is_const0()  # True if generates '0'
is_const1 = inst.is_const1()  # True if generates '1'
is_assign = inst.is_assign()  # True if assign (unnamed)
```

### Timing/Combinatorial Metadata
```python
has_timing = inst.has_modeling()  # True if has timing model

# Clock-related terms
inst.add_clock_related_inputs(clk_term, [input_term1, input_term2])
# Mark input_term1, input_term2 as related to clk_term
inst.add_clock_related_outputs(clk_term, [output_term1])
# Mark output_term1 as related to clk_term

# Combinatorial paths
inst.add_combinatorial_arcs([input1, input2], [output1])
# Mark input1, input2 as combinatorial inputs for output1
```

**Methods:**
- `add_clock_related_inputs(clock_term, input_terms)` - Register input terms that depend on a clock signal
- `add_clock_related_outputs(clock_term, output_terms)` - Register output terms that depend on a clock signal
- `add_combinatorial_arcs(input_terms, output_terms)` - Mark combinatorial paths from inputs to outputs

### Attributes
```python
attrs = list(inst.get_attributes())  # iterator over attributes
for attr in attrs:
    name = attr.get_name()
    value = attr.get_value() if attr.has_value() else None
```

### Export/Visualization
```python
inst.dump_verilog("output.v")                    # Verilog export
inst.dump_verilog("output.v", config=config)    # with config
inst.dump_full_dot("hierarchy.dot")              # full hierarchy DOT diagram
inst.dump_context_dot("context.dot")             # context/flat DOT diagram

truth_table = inst.get_truth_table()  # list[int] or None
```

The returned list is `[input_count, chunk0, chunk1, ...]`. Truth-table bits are
packed into 64-bit chunks. Assignment `n` is bit `n % 64` of chunk `n // 64`,
and input bit `i` is bit `i` of the assignment index. Prefer the output-specific
`Term.get_truth_table()` for multi-output cells.

**Export Methods:**
- `dump_verilog(path, config=None)` - Export instance to Verilog (.v file)
- `dump_full_dot(path)` - Export complete hierarchical structure to DOT format
- `dump_context_dot(path)` - Export flattened/context view to DOT format (useful for visualizing gate-level logic)

### Modification
```python
inst.set_name("new_instance_name")
inst.delete()  # delete this instance (recursive)
```

---

## Part 4: Term Class (Methods)

### Term Overview

In najaeda, a `najaeda.netlist.Term` (also referred to as a Port in Verilog terminology) can represent three scenarios:

1. **Single Bit Scalar Terminal** - A terminal representing a single scalar signal (width=1)
   - Example: `clk`, `reset`, `data_out`
   - Use `is_scalar()` to test

2. **Full Bus Terminal** - A terminal representing an entire bus (width > 1)
   - Example: `addr[31:0]`, `data[7:0]`
   - Use `is_bus()` to test
   - Access bits via `get_bit(index)` or iterate with `get_bits()`

3. **Single Bus Bit Terminal** - A terminal representing a single bit of a bus
   - Example: `addr[5]`, individual bit of a bus
   - Use `is_bus_bit()` to test
   - Get parent bus width with `get_width()`, index with `get_bit_number()`

### Term Direction Enum

```python
from najaeda.netlist import Term

# Term.Direction values
Term.Direction.INPUT      # Input terminal
Term.Direction.OUTPUT     # Output terminal
Term.Direction.INOUT      # Bidirectional terminal
```

---

### Basic Properties
```python
name = term.get_name()                 # term name
direction = term.get_direction()       # Term.Direction.INPUT, OUTPUT, INOUT
instance = term.get_instance()         # owning instance

# Position in bus
bit_index = term.get_bit_number()      # bit index if bus bit, else None
msb = term.get_msb()                   # MSB if bus
lsb = term.get_lsb()                   # LSB if bus
width = term.get_width()               # 1 if scalar, width if bus

equipotential = term.get_equipotential()  # get equipotential
```

### Type Tests
```python
is_scalar = term.is_scalar()       # True if scalar (not part of bus)
is_bit = term.is_bit()             # True if single bit
is_bus = term.is_bus()             # True if bus
is_bus_bit = term.is_bus_bit()     # True if bit of a bus
is_unnamed = term.is_unnamed()     # True if unnamed

is_input = term.is_input()         # True if input
is_output = term.is_output()       # True if output
is_sequential = term.is_sequential()  # True if sequential term
```

### Net Connectivity
```python
lower_net = term.get_lower_net()          # lower hierarchical net (child instance side)
upper_net = term.get_upper_net()          # upper hierarchical net (parent instance side)
                                          # Note: top-level terms return None for upper_net

term.connect_lower_net(net)               # connect to lower net (child hierarchy)
term.connect_upper_net(net)               # connect to upper net (parent hierarchy)
term.disconnect_lower_net()               # disconnect from lower
term.disconnect_upper_net()               # disconnect from upper
```

**Hierarchical Connection Model:**
- **Lower net**: Net connected on the inside of the hierarchical boundary (child instance)
- **Upper net**: Net connected on the outside of the hierarchical boundary (parent instance)
- Top-level terminals only have lower nets (no parent above)

### Fanout Analysis
```python
# Count fanout
fanout_count = term.count_flat_fanout()           # total fanout count
fanout_count = term.count_flat_fanout(filter=None)  # with optional filter

# Get fanout as list
fanout_list = term.get_flat_fanout()              # all fanout terms
fanout_list = term.get_flat_fanout(filter=None)   # with optional filter
```

**Notes:**
- `count_flat_fanout()` counts the number of sinks driven by this term
- `get_flat_fanout()` returns the actual fanout terms
- Filter parameter allows filtering fanout by instance type (optional)

### Combinatorial & Clock Analysis
```python
# Clock relationships
clock_inputs = term.get_clock_related_inputs()    # inputs related to this clock term
clock_outputs = term.get_clock_related_outputs()  # outputs related to this clock term

# Combinatorial paths
comb_inputs = term.get_combinatorial_inputs()     # combinatorial input terms
comb_outputs = term.get_combinatorial_outputs()   # combinatorial output terms
```

**Usage:**
- Call `get_clock_related_inputs()` on a clock term to find all inputs synchronized to it
- Call `get_clock_related_outputs()` on a clock term to find all outputs synchronized to it
- `get_combinatorial_inputs()` on an output term returns inputs with combinatorial paths to it
- `get_combinatorial_outputs()` on an input term returns outputs driven combinatorially by it

### Bit Access
```python
bits = list(term.get_bits())   # iterate all bits
bit = term.get_bit(index)      # get bit at index
```

### Attributes
```python
attrs = list(term.get_attributes())
count = term.count_attributes()
```

### Truth Table
```python
truth_table = term.get_truth_table()  # list[int] or None
```

**Notes:**
- Only output terms of combinatorial gate instances are valid; input terms raise
  `ValueError`
- Returns `None` when no truth table is modeled
- Otherwise returns `[input_count, chunk0, chunk1, ...]`, where each chunk packs
  64 output bits
- Assignment `n` uses bit `n % 64` of chunk `n // 64`; input term index `i`
  supplies assignment bit `i`
- Wrap `get_input_bit_terms()` in `list()` and preserve its order when mapping
  inputs to assignment bits

### Advanced (SNL Layer)
```python
snl_term = term.get_snl_term()  # access underlying SNL term (lower-level API)
key = term.key()                # unique, hashable identity (stable within design revision)
```

**Notes:**
- `get_snl_term()` provides access to the underlying SNL (Scalable Netlist) layer term
- Only use if working with SNL-level APIs directly
- `key()` returns a stable hashable tuple for use in dictionaries/sets across design sessions

## term.key() — quand l'utiliser et quand ne pas l'utiliser

UTILISER pour identifier un Term de façon stable :
```python
if term.key() in seen_keys: ...  # ok
```

NE PAS UTILISER pour dédoublonner dans un BFS — utiliser `term in visited` directement :
```python
# CORRECT
visited = set()
if term not in visited:
  visited.add(term)

# INCORRECT — term.key() identifie le term, pas l'équipotentiel
visited_keys = set()
if term.key() not in visited_keys:
  visited_keys.add(term.key())
```

### Modification
```python
term.set_msb(msb)
term.set_lsb(lsb)
term.delete()  # delete this term
```

---

## Part 5: Net Class (Methods)

### Net Overview

In najaeda, a `najaeda.netlist.Net` can represent four scenarios:

1. **Simple Scalar Net** - A net representing a single scalar signal (width=1)
   - Example: `clk`, `reset`, `enable`
   - Use `is_scalar()` to test

2. **Full Bus Net** - A net encompassing an entire bus (width > 1)
   - Example: `addr[31:0]`, `data[7:0]`
   - Use `is_bus()` to test
   - Access individual bits via `get_bit(index)` or iterate with `get_bits()`

3. **Single Bit of a Bus** - A net corresponding to an individual bit within a bus
   - Example: `addr[5]`, individual bit of a bus net
   - Use `is_bus_bit()` to test

4. **Concatenation of Bits** - A net created by combining multiple bits
   - Example: Result of `{bit1, bit2, bit3}` concatenation
   - Use `is_concat()` to test
   - Useful for modeling dynamic bit groupings

### Net Type Enum

```python
from najaeda.netlist import Net

# Net.Type enum values
Net.Type.STANDARD      # Regular signal (default)
Net.Type.ASSIGN0       # Verilog constant assignment 0
Net.Type.ASSIGN1       # Verilog constant assignment 1
Net.Type.ASSIGNX       # Verilog constant assignment X
Net.Type.ASSIGNZ       # Verilog constant assignment Z
Net.Type.SUPPLY0       # Constant-zero supply net
Net.Type.SUPPLY1       # Constant-one supply net
```

---

### Basic Properties
```python
name = net.get_name()          # net name
width = net.get_width()        # 1 if scalar, width if bus
msb = net.get_msb()            # MSB if bus
lsb = net.get_lsb()            # LSB if bus
```

### Type Tests
```python
is_scalar = net.is_scalar()    # True if scalar (width=1)
is_bit = net.is_bit()          # True if single bit
is_bus = net.is_bus()          # True if bus (width > 1)
is_bus_bit = net.is_bus_bit()  # True if single bit of a bus
is_concat = net.is_concat()    # True if concatenation

is_const = net.is_const()      # True if constant generator
is_const0 = net.is_const0()    # True if constant 0
is_const1 = net.is_const1()    # True if constant 1
```

**Notes:**
- `is_concat()` identifies nets created by bit concatenation operations
- Constant nets are generated by constant instance sources (driven by buffers with const values)

### Terminal Access
```python
# Get all terminals connected to this net
all_terms = list(net.get_terms())           # both design and instance terminals

# Get instance terminals only (internal connections to instances)
inst_terms = list(net.get_inst_terms())     # only instance-side terminals
n_inst = net.count_inst_terms()             # count of instance terminals

# Get design terminals only (top-level ports/interface)
design_terms = list(net.get_design_terms()) # only design-side terminals  
n_design = net.count_design_terms()         # count of design terminals
```

**Hierarchical Terminal Types:**
- **Design terminals:** Ports/interfaces of the current design (top-level connections)
- **Instance terminals:** Connections to child instances (internal connections)
- **All terminals:** Union of both types
- Note: `get_terms()` returns terminals bit-by-bit for bus nets

### Bus Bit Access
```python
bits = list(net.get_bits())   # iterate all bits (or self if scalar)
bit = net.get_bit(index)      # get bit at index
```

### Type Setting
```python
# Set net type
net.set_type(Net.Type.STANDARD)  # Regular signal (default)
net.set_type(Net.Type.ASSIGN0)   # Constant assignment 0
net.set_type(Net.Type.ASSIGN1)   # Constant assignment 1
net.set_type(Net.Type.ASSIGNX)   # Constant assignment X
net.set_type(Net.Type.ASSIGNZ)   # Constant assignment Z
net.set_type(Net.Type.SUPPLY0)   # Constant-zero supply
net.set_type(Net.Type.SUPPLY1)   # Constant-one supply
```

**Usage:**
- `STANDARD` is a regular interconnect net
- `ASSIGN0`, `ASSIGN1`, `ASSIGNX`, and `ASSIGNZ` represent Verilog assignment
  constants
- `SUPPLY0` and `SUPPLY1` represent constant supply nets and are appropriate
  when a created `logic0` or `logic1` cell drives the net
- Type information helps downstream tools understand power distribution

### Attributes
```python
attrs = list(net.get_attributes())
count = net.count_attributes()
```

### Advanced
```python
key = net.key()  # stable, hashable identity (unique within design revision)
```

**Notes:**
- `key()` returns a stable hashable tuple that uniquely identifies this net
- Use for storing nets in dictionaries or sets across design iterations
- Remains consistent within a single design revision (survives modifications that don't change net structure)

### Modification
```python
net.set_name("new_name")
net.set_msb(new_msb)
net.set_lsb(new_lsb)
net.delete()  # delete this net
```

---

## Part 6: Equipotential Class (Methods)

### Equipotential Overview

The `Equipotential` class manages equipotentials in the najaeda system. An equipotential represents a logically connected group of nets, terminals, and subcircuits at the same electrical potential.

**Key Concept:** Equipotentials allow analysis of signal connectivity across the design hierarchy, including:
- Finding all drivers and readers of a signal
- Detecting constant-driven nets
- Analyzing power/ground distribution

**Note:** Equipotential is a wrapper around SNL occurrence API for easier hierarchical navigation and analysis.

⚠️ Equipotential n'est PAS hashable (pas de `__hash__`).
Pour un BFS, dédoublonner directement sur `term` avec `term in visited`.
Utiliser `term.key()` uniquement quand il faut un identifiant stable du Term, pas pour la boucle de visite :

```python
visited = set()
for term in inst.get_input_bit_terms():
  if term not in visited:
    visited.add(term)
    queue.append(term.get_equipotential())
```

---

### Terminal Access
```python
# Instance-side terminals (connections within instances)
inst_terms = list(equip.get_inst_terms())   # instance terminals of this equipotential

# Top-level/design terminals (ports/interfaces)
top_terms = list(equip.get_top_terms())     # top-level terminals of this equipotential
```

**Hierarchical Terminal Types:**
- **Instance terminals** (`get_inst_terms()`) - Terminals connected to child instances (internal hierarchy)
- **Top terminals** (`get_top_terms()`) - Terminals at design/top level (ports and interfaces)

### Driver/Reader Analysis
```python
# Leaf-level drivers (at primitive instance level)
leaf_drivers = list(equip.get_leaf_drivers())            # all leaf drivers
leaf_drivers = list(equip.get_leaf_drivers(filter=None)) # with optional filter

# Leaf-level readers (at primitive instance level)
leaf_readers = list(equip.get_leaf_readers())            # all leaf readers
leaf_readers = list(equip.get_leaf_readers(filter=None)) # with optional filter

# Top-level drivers (at design/top level)
top_drivers = list(equip.get_top_drivers())              # top-level drivers

# Top-level readers (at design/top level)
top_readers = list(equip.get_top_readers())              # top-level readers
```

**Driver/Reader Levels:**
- **Leaf drivers/readers** - Analysis at the primitive gate level (gates and leaf cells)
  - `filter` parameter allows filtering by instance type (optional)
  - Returns **Term** objects (output/input terminals of leaf instances)
  - Call `.get_instance()` on each returned Term to get the owning Instance
  
- **Top drivers/readers** - Analysis at design/top level
  - Identifies top-level ports and interfaces as drivers/readers
  - Useful for I/O and power distribution analysis

**Common Patterns:**
```python
# Find who drives a signal at leaf level
drivers = equip.get_leaf_drivers()
if drivers:
  instances = [driver_term.get_instance() for driver_term in drivers]
  print(f"Signal driven by: {[inst.get_name() for inst in instances]}")

# Correct pattern: get_leaf_drivers() -> Term -> .get_instance() -> Instance
for driver_term in equip.get_leaf_drivers():
  inst = driver_term.get_instance()   # obligatoire
  inst.get_input_bit_terms()

# Find primary I/O readers at top level
io_readers = equip.get_top_readers()
```

### Constant Tests
```python
is_const = equip.is_const()    # True if constant generator
is_const0 = equip.is_const0()  # True if constant 0 generator
is_const1 = equip.is_const1()  # True if constant 1 generator
```

**Usage:**
- Use to detect constant-driven signals in design
- Useful for identifying redundant logic or optimization opportunities
- `is_const()` returns True if driven by any constant source
- `is_const0()` and `is_const1()` distinguish which constant value

**Example:**
```python
# Find all constant-driven nets in design
for equip in design_equipotentials:
    if equip.is_const():
        value = "0" if equip.is_const0() else "1"
        print(f"Signal always driven to {value}")
```

### Visualization
```python
equip.dump_dot("equip.dot")  # Export equipotential connectivity to DOT format
```

**Purpose:**
- Generates a Graphviz DOT file visualizing the equipotential structure
- Shows all terminals, drivers, readers, and their interconnections
- Useful for debugging signal connectivity and finding routing issues

**Output File:**
- Format: Graphviz DOT (.dot)
- Can be converted to image: `dot -Tpng equip.dot -o equip.png`
- Displays hierarchy of connections at all levels (leaf to top)

---

## Part 7: Attribute Class (Methods)

### Attribute Overview

The `Attribute` class wraps metadata attributes attached to design objects (Instances, Terms, Nets).

**Key Concept:** Attributes provide extensible key-value storage for:
- RTL annotations
- Tool-specific metadata
- Design properties and constraints
- Custom EDA flow information

Attributes are attached to instances, terms, and nets via `get_attributes()` iterators.

---

### Properties
```python
name = attr.get_name()         # attribute name (str)
value = attr.get_value()       # attribute value (str)
has_val = attr.has_value()     # True if attribute has a value (bool)
```

### Typical Usage
```python
# Iterate over instance attributes
for attr in inst.get_attributes():
    name = attr.get_name()
    if attr.has_value():
        value = attr.get_value()
        print(f"  {name} = {value}")
    else:
        print(f"  {name} (no value)")

# Find specific attribute
for attr in net.get_attributes():
    if attr.get_name() == "preserve":
        print(f"Net is marked for preservation")
```

**Notes:**
- All attribute values are stored as strings
- Use `has_value()` to distinguish valueless attributes from those with empty strings
- Attributes are typically read-only (set via configuration or Verilog attributes)

---

## Part 8: Module-Level Functions

### Navigation
```python
top = netlist.get_top()  # Get top-level instance
```

### Instance Location
```python
inst = netlist.get_instance_by_path(["top", "cpu"])  # by path list
inst = netlist.get_instance_by_path("top/cpu")       # also string with "/" separator
```

**Notes:**
- `get_instance_by_path(names: list)` → Instance or None
- Accepts both list format `["top", "cpu"]` and string format `"top/cpu"`
- Returns None if path does not exist in hierarchy

### Design Statistics
```python
max_fanout = netlist.get_max_fanout()         # maximum fanout (list of instances)
max_logic_level = netlist.get_max_logic_level()  # max combinatorial depth (list of instances)
```

**Return Types:**
- `get_max_fanout()` returns `list` - Instances with maximum fanout
- `get_max_logic_level()` returns `list` - Instances with maximum logic level

### Model Information
```python
model_name = netlist.get_model_name(id_tuple)  # lookup model by ID
lib = netlist.get_primitives_library()         # get embedded primitives library (NLLibrary)
```

**Notes:**
- `get_model_name(id: tuple[int, int, int])` → str or None
  - Looks up model name by its 3-tuple ID (library_id, cell_id, view_id)
  - Returns None if ID not found
  
- `get_primitives_library()` → najaeda.naja.NLLibrary
  - Returns the current embedded primitives library
  - Only available if `load_primitives()` was called

### ID Resolution (Advanced SNL Layer)
```python
inst = netlist.get_snl_instance_from_id_list([1, 2, 3])
path = netlist.get_snl_path_from_id_list([1, 2])
term = netlist.get_snl_term_for_ids_with_path(path, term_id, bit_index)
```

### Transformation
```python
netlist.apply_dle()  # Dead Logic Elimination (DLE) - removes unused logic
netlist.apply_constant_propagation()  # Constant propagation - replaces signals with constant values
```

**Transformation Functions:**
- `apply_dle()` → None - Apply Dead Logic Elimination to the top design
  - Removes instances and nets that are not connected to outputs
  - Useful for optimization after synthesis or netlist modifications
  
- `apply_constant_propagation()` → None - Apply constant propagation to the top design
  - Replaces constant-driven signals with their constant values
  - Removes redundant logic depending on constants
  - **Warning:** Can segfault on certain designs; use in separate process if critical

### Export/Save
```python
netlist.dump_naja_if("design.naja")  # Save in Naja IF format
```

### Utilities
```python
hash_val = netlist.consistent_hash(obj)  # stable hash for object
```

**Notes:**
- `consistent_hash(obj)` → hash value (int)
  - Generates a stable, deterministic hash for any netlist object (Instance, Term, Net, etc.)
  - Useful for comparing designs across different sessions
  - Two designs with identical structure will generate identical hashes

### Environment
```python
netlist.create_top("top_name")  # Create new empty top instance
netlist.reset()  # Reset entire environment
```

**Notes:**
- `create_top(name: str)` → Instance
  - Creates a fresh top-level instance with given name
  - Useful for starting from scratch
  
- `reset()` → None
  - Clears all designs, libraries, and instances from memory
  - Resets the Naja environment to initial state
  - Useful between multiple independent design analyses

---

## Part 9: Instance Visitor Module

**Import:**
```python
from najaeda import instance_visitor
```

### VisitorConfig (class)
Configuration for instance traversal with callbacks
```python
def my_callback(instance):
    print(f"Visiting {instance.get_name()}")
    return None  # return None to continue, or value to stop

config = instance_visitor.VisitorConfig(
    callback=my_callback  # function(instance) -> None/value
)
instance_visitor.visit(top, config)
```

### VisitorConfig with User Arguments
```python
def count_callback(instance, counter_dict):
    counter_dict["count"] += 1

counter = {"count": 0}
config = instance_visitor.VisitorConfig(
    callback=count_callback,
    args=(counter,)  # tuple of additional arguments
)
instance_visitor.visit(top, config)
print(f"Total instances: {counter['count']}")
```

### visit() Function
```python
instance_visitor.visit(instance, config)  # Traverse with config
```

---

## Part 10: Stats Module

**Import:**
```python
from najaeda import stats
```

### Design Statistics Export
```python
import najaeda.stats as stats

# Write statistics to file
with open("design.stats", "w") as f:
    stats.dump_instance_stats_text(top, f)
```

**Output:** Text report with detailed statistics per module:
- Instance count
- Net count
- Terminal count
- Hierarchy depth

---

## Part 11: Common Error Patterns

| Operation | Common Error | Solution |
|-----------|-------------|----------|
| `load_verilog()` | FileNotFoundError | Check path exists and is readable |
| `get_instance_by_path()` | Returns None | Verify hierarchical path structure |
| `connect()` | TypeError: width mismatch | Ensure net width matches terminal width |
| `delete_net()` | RuntimeError: still connected | Call `disconnect()` first |
| `get_input_bit_terms()` | Can't index | Wrap with `list()`: `list(inst.get_input_bit_terms())` |
| Export `dump_verilog()` | ValueError: not .v | Ensure path ends with `.v` |
| Export `dump_verilog()` | FileNotFoundError | Ensure parent directory exists |

---

## Part 12: Usage Pattern (Complete Script)

<!-- example-purpose: standalone script | complete script | visitor | load libraries | load design -->

```python
from pathlib import Path
import logging

import najaeda.netlist as netlist
from najaeda import instance_visitor

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Load design
design_dir = Path("design_dir").resolve()
netlist.load_liberty([str(f) for f in design_dir.glob("*.lib")])
top = netlist.load_verilog(
    design_dir / "circuit.v",
    config=netlist.VerilogConfig(keep_assigns=True, allow_unknown_designs=True),
)
if top is None:
    logger.error("Failed to load design")
    exit(1)

# Navigate
inst = top.get_instance_by_path("top/module/submodule")
if inst is None:
    logger.error("Instance not found")
    exit(1)

# Query
inputs = list(inst.get_input_bit_terms())
outputs = list(inst.get_output_bit_terms())

# Modify
net = top.create_net("new_signal")
if inputs:
    inputs[0].connect_lower_net(net)

# Transform
netlist.apply_dle()
netlist.apply_constant_propagation()

# Visitor pattern
def print_all(inst):
    logger.info(f"{inst.get_name()}: {inst.get_model_name()}")

config = instance_visitor.VisitorConfig(callback=print_all)
instance_visitor.visit(top, config)

# Export
top.dump_verilog("output.v")
netlist.dump_naja_if("design.naja")
logger.info("Done")
```

---

## Part 13: Function Loading Notes

**Loading Guidance:**
- `load_primitives()` is mandatory for Xilinx FPGA netlists. Call `netlist.load_primitives('xilinx')` first and treat any failure as fatal.
- For non-Xilinx flows on builds where `load_primitives()` is unavailable, do not invent a fallback API.

- `apply_constant_propagation()` can segfault on certain designs. Use in separate process if critical.

---

**For forbidden/deprecated API list, see [resources/api-rules.md](resources/api-rules.md).**
