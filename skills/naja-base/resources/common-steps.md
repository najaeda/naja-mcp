# Common Steps: Patterns for Naja Scripts

All patterns use only verified functions from najaeda.netlist.

## Règle avant d'écrire tout script

1. Copier le pattern le plus proche de [working-examples.md](working-examples.md)
2. Ne modifier QUE ce qui est nécessaire pour le cas demandé
3. Ne jamais inventer une API — si elle n'est pas dans [api-functions.md](api-functions.md)
    ou [working-examples.md](working-examples.md), elle n'existe probablement pas
4. Les types de retour réels sont dans [working-examples.md](working-examples.md),
    pas dans la doc officielle

**Module:** `import najaeda.netlist as netlist`

---

## Pattern BFS NajaEDA — validé sur vexriscv et black_parrot

```python
# Dans __main__ :
instances = set()   # GLOBAL — accessible depuis la fonction

# Dans la fonction :
visited = set()     # TOUJOURS set(), jamais list()
queue   = deque()

for term in top.get_output_bit_terms():
    if term not in visited:          # dédoublonner AVANT
        visited.add(term)
        queue.append(term)

while queue:
    term = queue.popleft()
    equip = term.get_equipotential()
    if equip is None:
        continue
    for driver_term in equip.get_leaf_drivers():   # retourne Term, pas Instance
        inst = driver_term.get_instance()           # .get_instance() obligatoire
        if inst not in instances:
            instances.add(inst)
            for in_term in inst.get_input_bit_terms():
                if in_term not in visited:           # dédoublonner AVANT
                    visited.add(in_term)
                    queue.append(in_term)
```

Règles :
- `set()` partout — jamais `list()` pour les collections visitées
- `get_leaf_drivers()` -> `Term` -> `.get_instance()` -> `Instance`
- Dédoublonner AVANT d'ajouter dans la queue, pas après
- `instances` défini dans `__main__`, pas dans la fonction

---

## Pattern 1: Load libs, then Verilog

```python
from pathlib import Path
import najaeda.netlist as netlist

input_path = Path("circuit.v")
design_dir = input_path.resolve().parent

is_xilinx_fpga_netlist = False  # set True only for Xilinx FPGA designs

if is_xilinx_fpga_netlist:
    try:
        netlist.load_primitives('xilinx')
    except Exception as exc:
        logger.error(f"Failed to load Xilinx primitives: {exc}")
        return 1

netlist.load_liberty([str(f) for f in Path(design_dir).glob("*.lib")])
top = netlist.load_verilog(
    str(input_path),
    config=netlist.VerilogConfig(keep_assigns=True, allow_unknown_designs=True),
)
if top is None:
    raise SystemExit(1)
```

Règle:
- `load_primitives('xilinx')` only for Xilinx FPGA netlists
- standard ASIC / Liberty flows: load `.lib` files, then `load_verilog()`

## Pattern 2b: Discover usable nets before editing

```python
named_nets = [n for n in top.get_nets() if n.get_name()]

for net in named_nets:
    terms = list(net.get_terms())
    has_primary_output = any(t.is_output() and t.get_instance().is_top() for t in terms)
    has_sequential_fanout = any(t.get_instance().is_sequential() for t in terms)
    if has_primary_output or has_sequential_fanout:
        print(net.get_name())
```

Règles:
- Ne jamais supposer que `list(top.get_nets())[0]` est un net utile
- Filtrer d'abord les nets nommés
- Privilégier un net visible par une sortie primaire ou un élément séquentiel si la modification doit être détectable par Kepler / ECO / vérification formelle

Note Kepler:
- If the change does not reach a primary output or a DFF input, Kepler may not surface it
- anonymous or internal-only nets are usually poor verification targets
---

## Pattern 2: Query connectivity safely (generator APIs)

```python
leaves = list(top.get_leaf_children())
for leaf in leaves:
    in_terms = list(leaf.get_input_bit_terms())
    out_terms = list(leaf.get_output_bit_terms())
    print(f"{leaf.get_model_name()}: {len(in_terms)} inputs, {len(out_terms)} outputs")
```

---

## Building Block: Materialize and count API collections

NajaEDA collection APIs such as `get_leaf_children()`,
`get_input_bit_terms()`, and `get_output_bit_terms()` return iterables or
generators. Materialize the requested collection before counting or retaining
it as an analysis result:

```python
leaf_instances = list(top.get_leaf_children())
leaf_count = len(leaf_instances)

input_terms = list(top.get_input_bit_terms())
input_count = len(input_terms)

output_terms = list(top.get_output_bit_terms())
output_count = len(output_terms)
```

Choose the collection from the requested metric; do not substitute a different
term, net, or instance count merely because it is also available. When an
integration contract requests named helper functions, include the complete
`def` block for every helper the generated script calls, including transitive
helpers. Skill snippets are documentation and are not imported into a generated
script at runtime. A call such as `measure_metric(top)` is executable only when
that same generated script defines `measure_metric`.

---

## Building Block: Backward dependency cones

<!-- example-purpose: backward | fan in | liveness | observability | loadless | dead logic | dle | dependency cone -->

Backward traversal from observable or otherwise important sink terms is a
reusable primitive for fan-in reporting, design slicing, liveness analysis,
observability analysis, and impact analysis. Follow each term's equipotential
to its leaf drivers, retain each driver's owning instance, and continue from
that instance's input terms:

```python
def collect_backward_leaf_instances(seed_terms):
    retained_instances = set()
    visited_term_keys = set()
    pending_terms = list(seed_terms)

    while pending_terms:
        term = pending_terms.pop()
        term_key = term.key()
        if term_key in visited_term_keys:
            continue
        visited_term_keys.add(term_key)

        equipotential = term.get_equipotential()
        for driver_term in list(equipotential.get_leaf_drivers()):
            driver_instance = driver_term.get_instance()
            if driver_instance in retained_instances:
                continue
            retained_instances.add(driver_instance)
            pending_terms.extend(list(driver_instance.get_input_bit_terms()))

    return retained_instances
```

The caller chooses the seeds and therefore defines what is observable or
important. Top-level outputs can be seeded with
`list(top.get_output_bit_terms())`. Cells with no output terms can represent
sinks or side effects; when a policy must preserve them, add their input terms
to the seed list. This helper only computes a retained set and performs no edit.

## Building Block: Snapshot-based object deletion

Mutation invalidates live hierarchy iterators and object handles. First
materialize the candidates and finish all analysis, then delete the chosen
instances from that snapshot:

```python
def instances_with_output_terms(instances):
    return [
        instance
        for instance in list(instances)
        if list(instance.get_output_bit_terms())
    ]


def delete_instances(instances):
    for instance in list(instances):
        instance.delete()
```

`instances_with_output_terms()` is a structural classification helper, not a
mutation policy; it is useful whenever an analysis needs to distinguish drivers
from sink-only or side-effect cells. Deletion belongs to the `Instance` itself
through `instance.delete()`. There is no `delete_instance()` method on an
instance or its parent. Candidate selection is a separate policy: a caller can
subtract a retained dependency cone from
`list(top.get_leaf_children())`, filter by output terms or attributes, or apply
another analysis before passing objects to this helper.

---

## Pattern 3: Modify and reconnect

```python
leaves = list(top.get_leaf_children())
if not leaves:
    raise SystemExit("No leaf cells")

candidate = leaves[0]
in_terms = list(candidate.get_input_bit_terms())

if in_terms:
    new_net = top.create_net("new_signal")
    in_terms[0].disconnect_upper_net()  # disconnect first on the parent-facing side
    in_terms[0].connect_upper_net(new_net)  # then reconnect
```

### Mini workflow: connect, move, and insert

```python
# a) Insert a child instance and connect it
child = top.create_child_instance("my_cell", "u0")
din = child.get_term("A")
dout = child.get_term("Y")
din.connect_upper_net(top.get_net("src_net"))
dout.connect_upper_net(top.create_net("mid_net"))

# b) Disconnect a term and reconnect it elsewhere
term = child.get_term("A")
term.disconnect_upper_net()
term.connect_upper_net(top.get_net("alt_src_net"))

# c) Insert a cell in series on a logical path
path_net = top.get_net("logic_path")
mid = top.create_net("logic_path_mid")
buf = top.create_child_instance("BUF_X1", "u_buf")
buf.get_term("A").disconnect_upper_net()
buf.get_term("A").connect_upper_net(path_net)
buf.get_term("Y").connect_upper_net(mid)
```

Note: when the documentation asks for a lower-side operation or when you are manipulating the top-level view, use `connect_lower_net()` / `disconnect_lower_net()` instead.

---

## Building Block: Packed truth-table evaluation

<!-- example-purpose: truth table | truth tables | constant propagation | boolean analysis | simulation | pattern matching -->

Truth tables are useful for many Boolean analyses, including simulation,
dependency checks, simplification, and pattern matching. The first list element
is the input count; the remaining elements are packed 64-bit output chunks.

```python
def truth_value(truth_table, assignment):
    chunk_index = 1 + assignment // 64
    bit_index = assignment % 64
    return (truth_table[chunk_index] >> bit_index) & 1


def compatible_assignments(input_count, fixed_inputs):
    unknown = [
        index for index in range(input_count) if index not in fixed_inputs
    ]
    for unknown_assignment in range(1 << len(unknown)):
        assignment = 0
        for index, value in fixed_inputs.items():
            assignment |= value << index
        for offset, index in enumerate(unknown):
            value = (unknown_assignment >> offset) & 1
            assignment |= value << index
        yield assignment
```

Use `output_term.get_truth_table()` only on scalar output terms. Preserve the
order from `list(instance.get_input_bit_terms())`: input term index `i` is bit
`i` of an assignment. A `None` truth table means the cell has no usable Boolean
model.

## Building Block: Known constant inputs

<!-- example-purpose: constant propagation | constant inputs | constant input | constant folding -->

Equipotential predicates identify constants independently of whether they came
from a literal, assignment, supply net, or constant cell.

```python
def known_constant_inputs(input_terms):
    fixed_inputs = {}
    for index, input_term in enumerate(input_terms):
        equipotential = input_term.get_equipotential()
        if equipotential.is_const0():
            fixed_inputs[index] = 0
        elif equipotential.is_const1():
            fixed_inputs[index] = 1
    return fixed_inputs
```

This helper reports only known inputs. The calling algorithm decides how to use
partial assignments and whether an observed property is strong enough to edit
the design.

## Building Block: Possible values under partial inputs

<!-- example-purpose: constant propagation | partial inputs | possible values | truth table | constant folding -->

Partial truth-table evaluation is a reusable analysis primitive for simulation,
Boolean dependency analysis, observability, simplification, ECO planning, and
repair. It reports every output value compatible with the inputs currently
known to be constant, without deciding what an algorithm should do with that
fact.

```python
def possible_output_values(output_term, fixed_inputs):
    truth_table = output_term.get_truth_table()
    if truth_table is None:
        return None
    input_count = truth_table[0]
    return {
        truth_value(truth_table, assignment)
        for assignment in compatible_assignments(input_count, fixed_inputs)
    }
```

To collect these facts across a flat combinational design, traverse instances
and terms through their owning objects:

```python
def collect_modeled_output_facts(top):
    facts = []
    for instance in list(top.get_leaf_children()):
        input_terms = list(instance.get_input_bit_terms())
        fixed_inputs = known_constant_inputs(input_terms)
        for output_term in list(instance.get_output_bit_terms()):
            possible_values = possible_output_values(output_term, fixed_inputs)
            if possible_values is not None:
                facts.append((output_term, possible_values))
    return facts
```

Each result is a two-item tuple `(output_term, possible_values)`, and
`possible_values` is a `set`. Unpack the tuple directly. Sets are not
subscriptable; when a consumer needs the sole value of a singleton set, use
`next(iter(possible_values))` only after checking its length:

```python
for output_term, possible_values in collect_modeled_output_facts(top):
    if len(possible_values) == 1:
        sole_value = next(iter(possible_values))
```

The returned facts are analysis results, not edit instructions. A caller may
use them for reporting, dependency checks, candidate selection, or a separately
defined mutation policy.

## Building Block: Constant sources and reader rewiring

<!-- example-purpose: constant propagation | constant source | constant sources | constant folding | tie cell | tie cells | rewire readers -->

A fresh constant source needs a typed parent net and a `logic0` or `logic1`
model from the loaded primitives or Liberty library. The caller supplies names
so the same building block can be used by optimization, ECO, instrumentation,
or repair algorithms.

```python
def create_constant_source(top, value, net_name, instance_name):
    constant_net = top.create_net(net_name)
    constant_net.set_type(
        netlist.Net.Type.SUPPLY1 if value else netlist.Net.Type.SUPPLY0
    )
    driver = top.create_child_instance(
        model=f"logic{value}",
        name=instance_name,
    )
    driver_output = list(driver.get_output_bit_terms())[0]
    driver_output.connect_upper_net(constant_net)
    return constant_net


def get_or_create_constant_source(
    top,
    value,
    cache,
    net_names,
    instance_names,
):
    if value not in cache:
        cache[value] = create_constant_source(
            top,
            value,
            net_names[value],
            instance_names[value],
        )
    return cache[value]


def rewire_equipotential_readers(equipotential, target_net):
    for reader in list(equipotential.get_leaf_readers()):
        reader.disconnect_upper_net()
        reader.connect_upper_net(target_net)
    for reader in list(equipotential.get_top_readers()):
        reader.disconnect_lower_net()
        reader.connect_lower_net(target_net)
```

The caller owns and defines the cache and naming maps. For example:

```python
constant_sources = {}
constant_net_names = {0: "logic0_naja_net", 1: "logic1_naja_net"}
constant_instance_names = {0: "logic0_naja", 1: "logic1_naja"}

target_net = get_or_create_constant_source(
    top,
    selected_value,
    constant_sources,
    constant_net_names,
    constant_instance_names,
)
```

The first argument to `rewire_equipotential_readers()` is an `Equipotential`,
not a `Term`. For a selected output term, call it as follows:

```python
rewire_equipotential_readers(
    output_term.get_equipotential(),
    target_net,
)
```

Collect edit candidates before rewiring. Do not mutate connectivity while
iterating a live design generator. Reuse a constant source when multiple edits
need the same value, and choose names that cannot collide with existing objects.
Create a shared source lazily, after an edit candidate actually needs its value;
do not add unused source cells or nets to the design. Keep the cache outside the
candidate loop so all edits requesting the same value reuse one source.

## Building Block: Compose analysis and mutation safely

This control-flow pattern applies to optimizers, ECOs, instrumentation, repair,
and other algorithms that derive edits from the current graph:

1. Materialize traversal generators with `list()`.
2. Analyze objects without changing connectivity.
3. Store each decision and the exact object handles needed to apply it.
4. Create shared resources once and cache them by purpose or value.
5. Apply the collected edits only after analysis finishes.
6. If edits can expose new candidates, repeat until one pass makes no changes.

Do not replace a requested algorithm with a similarly named native transform
when the request explicitly forbids that transform. Compose the documented
building blocks and keep computation separate from graph mutation.

---

## Pattern 4: Apply native transforms

```python
netlist.apply_dle()
netlist.apply_constant_propagation()
```

---

## Pattern 5: Export correctly

```python
from pathlib import Path

output_path = Path("output.v")
if output_path.suffix != ".v":
    raise SystemExit("Output path must end with .v")
if not output_path.parent.exists():
    raise SystemExit(f"Output directory does not exist: {output_path.parent}")

top.dump_verilog(str(output_path))
```

---

## Required Checks Before Running

- Input file exists and is readable
- For Xilinx FPGA netlists, load primitives first with `netlist.load_primitives('xilinx')`
- Liberty loading happens before Verilog loading
- top is not None after load_verilog
- list() wrapping for get_output_bit_terms/get_input_bit_terms/get_leaf_children
- Output path ends with .v
- Output directory already exists
- Named nets are filtered before any structural edit
- Upper/lower direction is chosen deliberately for every term operation

---

For the canonical list of forbidden or deprecated API calls, see [resources/api-rules.md](resources/api-rules.md).
