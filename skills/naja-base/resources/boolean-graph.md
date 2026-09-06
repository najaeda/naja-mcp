# Boolean graph collection

## Collect Boolean connectivity

<!-- example-purpose: constant propagation | collect_boolean_graph -->

Read-only: no propagation or edits. Returns (cells, consumers, seeds, readers):
- cells[i] = (ordered input IDs, [(output ID, packed table), ...]).
- consumers[ID] = cell indices; seeds[ID] = initial 0/1.
- readers[ID] = frozen eligible Terms, empty for unsafe signals.

IDs distinguish hierarchical occurrences and last until mutation.
Copy seeds into facts; missing facts are unknown. Preserve seed IDs when editing.

```python
def collect_boolean_graph(top):
    term_signals, safe, seeds, readers = {}, {}, {}, {}

    def constant_driver(output):
        owner = output.get_instance()
        if not output.is_output() or owner.is_sequential():
            return None
        table = output.get_truth_table()
        if table is not None and table[0] == 0 and len(table) >= 2:
            if not list(owner.get_input_bit_terms()):
                return table[1] & 1
        return None

    def signal(term):
        key = term.key()
        if key in term_signals:
            return term_signals[key]
        equip = term.get_equipotential()
        members = list(equip.get_inst_terms()) + list(equip.get_top_terms())
        identity = frozenset(member.key() for member in members) or frozenset([key])
        for member in members + [term]:
            term_signals[member.key()] = identity
        drivers = list(equip.get_leaf_drivers())
        ports = list(equip.get_top_drivers())
        bad = len(drivers) + len(ports) > 1
        bad = bad or any(not driver.is_output() for driver in drivers)
        bad = bad or any(not port.is_input() for port in ports)
        values = {v for v, yes in ((0, equip.is_const0()), (1, equip.is_const1())) if yes}
        bad = bad or equip.is_constx() or equip.is_constz()
        for member in members + [term]:
            net = (member.get_lower_net() if member.get_instance().is_top()
                   else member.get_upper_net())
            if net is not None:
                bad = bad or net.is_constx() or net.is_constz()
                if net.is_const0():
                    values.add(0)
                if net.is_const1():
                    values.add(1)
        value = constant_driver(drivers[0]) if len(drivers) == 1 else None
        if values:
            bad = bad or len(values) != 1 or bool(ports)
            if drivers:
                bad = bad or value not in values
            value = next(iter(values))
        safe[identity] = not bad
        if not bad and value is not None:
            seeds[identity] = value
        leaf_readers = [r for r in equip.get_leaf_readers() if r.is_input()]
        top_readers = [r for r in equip.get_top_readers() if r.is_output()]
        readers[identity] = leaf_readers + top_readers if not bad else []
        return identity

    cells, consumers = [], {}
    for instance in list(top.get_leaf_children()):
        inputs = list(instance.get_input_bit_terms())
        input_ids = [signal(term) for term in inputs]
        output_pairs = []
        for output in list(instance.get_output_bit_terms()):
            output_id = signal(output)
            if instance.is_sequential() or not output.is_output() or not safe[output_id]:
                continue
            table = output.get_truth_table()
            if table is None or table[0] != len(inputs):
                continue
            if any(not term.is_input() for term in inputs):
                continue
            if len(table) < 1 + ((1 << table[0]) + 63) // 64:
                continue
            output_pairs.append((output_id, table))
        if output_pairs:
            cell_index = len(cells)
            cells.append((input_ids, output_pairs))
            for input_id in set(input_ids):
                consumers.setdefault(input_id, []).append(cell_index)
    for term in list(top.get_output_bit_terms()):
        signal(term)
    return cells, consumers, seeds, readers
```
