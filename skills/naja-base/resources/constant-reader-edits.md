# Deferred constant-reader edits

## Apply already-proved reader constants

<!-- example-purpose: constant propagation | constant dataflow | worklist | constant reader edits | deferred reader edits | apply reader constants -->

`apply_reader_constants(top, edits)` accepts a snapshot of `(reader Term, 0/1)`.
It applies proved decisions only; it does not discover or propagate constants.
Supports leaf input bits and top output bits. The loaded library must provide
zero-input, single-output `logic0` and `logic1` models. No native optimization
is called. Existing valid connections remain unchanged; newly needed sources
are shared per occurrence/value and have modeled drivers for Verilog export.

```python
def apply_reader_constants(top, edits):
    decisions = {}
    for reader, value in list(edits):
        if value not in (0, 1):
            raise ValueError("Reader value must be binary")
        value = int(value)
        instance = reader.get_instance()
        if not ((instance.is_top() and reader.is_output() and not reader.is_bus())
                or (instance.is_leaf() and reader.is_input() and not reader.is_bus())):
            raise ValueError("Expected a leaf input bit or top output bit")
        if reader.key() in decisions and decisions[reader.key()][1] != value:
            raise ValueError("Conflicting reader edits")
        decisions[reader.key()] = (reader, value)
    cache = {}
    public_names = {term.get_name() for term in list(top.get_terms())}

    def current_net(reader):
        return reader.get_lower_net() if reader.get_instance().is_top() else reader.get_upper_net()

    def constant_info(net):
        if net is None or net.is_constx() or net.is_constz():
            return None, 0
        typed = 0 if net.is_const0() else (1 if net.is_const1() else None)
        terms = list(net.get_terms())
        drivers = []
        if terms:
            equipotential = terms[0].get_equipotential()
            drivers = list(equipotential.get_leaf_drivers())
            if list(equipotential.get_top_drivers()) or len(drivers) > 1:
                return None, len(drivers)
        if not drivers:
            return typed, 0
        driver = drivers[0]
        instance = driver.get_instance()
        table = driver.get_truth_table() if driver.is_output() else None
        if (instance.is_sequential() or list(instance.get_input_bit_terms())
                or table is None or table[0] != 0):
            return None, 1
        value = table[1] & 1
        return (value if typed is None or typed == value else None), 1

    def fresh_name(owner, base):
        name, suffix = base, 0
        local_names = {term.get_name() for term in list(owner.get_terms())}
        while (name in public_names or name in local_names or owner.get_net(name) is not None
               or owner.get_child_instance(name) is not None):
            suffix += 1
            name = base + "_" + str(suffix)
        return name

    def source_for(owner, owner_ids, value):
        key = (tuple(owner_ids), value)
        if key not in cache:
            name = None
            needs_driver = True
            for candidate in list(owner.get_nets()):
                if not candidate.is_scalar() or not candidate.get_name():
                    continue
                if candidate.get_name() in public_names:
                    continue
                typed = candidate.is_const1() if value else candidate.is_const0()
                known, count = constant_info(candidate)
                if typed and known == value and (count == 1 or not list(candidate.get_terms())):
                    name, needs_driver = candidate.get_name(), count == 0
                    break
            if name is None:
                name = fresh_name(owner, "logic" + str(value) + "_naja_net")
                owner.create_net(name)
            if needs_driver:
                cell_name = fresh_name(owner, "logic" + str(value) + "_naja")
                cell = owner.create_child_instance(model="logic" + str(value), name=cell_name)
                outputs = list(cell.get_output_bit_terms())
                if list(cell.get_input_bit_terms()) or len(outputs) != 1:
                    raise ValueError("Invalid constant source model")
                table = outputs[0].get_truth_table()
                if table is None or table[0] != 0 or (table[1] & 1) != value:
                    raise ValueError("Constant source model has wrong Boolean value")
                source = owner.get_net(name)
                source.set_type(netlist.Net.Type.SUPPLY1 if value else netlist.Net.Type.SUPPLY0)
                outputs[0].connect_upper_net(source)
            cache[key] = name
        return owner.get_net(cache[key])

    for reader, value in decisions.values():
        if constant_info(current_net(reader))[0] == value:
            continue
        owner_ids = list(reader.key()[0][:-1])
        owner = top.get_child_instance_by_id(owner_ids) if owner_ids else top
        source = source_for(owner, owner_ids, value)
        source_name = source.get_name()
        old = current_net(reader)
        if reader.get_instance().is_top():
            if old is not None and old.get_name() in public_names:
                parent_net = top.get_net(old.get_name()) if old.is_bus_bit() else old
                parent_net.set_name(fresh_name(top, old.get_name() + "_before_constant"))
            reader.disconnect_lower_net()
            reader.connect_lower_net(owner.get_net(source_name))
        else:
            reader.disconnect_upper_net()
            reader.connect_upper_net(owner.get_net(source_name))
```

Renaming the old scalar net or containing bus prevents the dumper reconnecting old
drivers to a moved public output by name. Source cache entries are names,
so nets are reacquired after occurrence uniquification. Untyped primitive nets
with existing readers are preserved rather than retagged. Ambiguous drivers
are never reused as sources. Top outputs are edited bitwise; whole-bus Terms
are rejected before mutation. A containing bus is renamed once, preserving
its unedited bits and declared bounds.
