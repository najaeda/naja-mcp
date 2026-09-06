# Gate Replacement: Preserve Boundary Connections

<!-- example-purpose: gate replacement | replace gates | replacement circuit | gate group | replace a group | replace two inverters -->

Use this pattern for a local combinational rewrite whose equivalent replacement
circuit is already specified. The example replaces two serial inverters with a
buffer. Cell models and pin names are illustrative; use the task's loaded
library, original instance names, replacement circuit, and complete pin mapping.

Capture every required boundary Net before mutation. An Instance name locates a
cell, a Term name locates a pin, and Term.get_upper_net() returns that pin's
parent-facing connection. Symbolic mapping keys are not design net names.
Preserve original output Net objects so all existing readers remain connected.

Keep symbolic boundary keys separate from literal library pin names. For
example, a mapping key such as "input" can refer to the Net attached to pin "A";
the key is never substituted for the pin name in get_term().
Use explicit Term variables when checking for missing pins before dereferencing
them. Direct and chained API calls are both supported by 22b. The example below
keeps these operations separate so each captured Term and Net can be checked.

```python
def edit(top):
    first = top.get_child_instance("old_first")
    second = top.get_child_instance("old_second")
    assert first is not None and second is not None
    assert first.get_model_name() == "INV_X1"
    assert second.get_model_name() == "INV_X1"
    assert top.get_child_instance("replacement") is None

    # Capture inputs, outputs, and internal connections before deleting anything.
    pins = {
        "input": first.get_term("A"),
        "middle_out": first.get_term("ZN"),
        "middle_in": second.get_term("A"),
        "output": second.get_term("ZN"),
    }
    wires = {}
    for key, term in pins.items():
        assert term is not None
        connection = term.get_upper_net()
        assert connection is not None
        wires[key] = connection
    assert wires["middle_out"] == wires["middle_in"]
    assert set(wires["middle_out"].get_terms()) == {
        pins["middle_out"], pins["middle_in"]
    }

    replacement = top.create_child_instance(
        model="BUF_X1", name="replacement"
    )
    for pin, key in (("A", "input"), ("Z", "output")):
        term = replacement.get_term(pin)
        assert term is not None
        term.connect_upper_net(wires[key])

    # Delete only after every replacement input and output is connected.
    first.delete()
    second.delete()
    return top
```

The intermediate-net check rejects extra readers or exposed ports that this
particular replacement would otherwise disconnect. Preserve such boundary
signals explicitly when the requested replacement includes them.

For a replacement with multiple gates, capture all boundary connections first,
then create and connect the specified replacement rows in dependency order.
Allocate new nets only for replacement-internal signals. Reuse captured Nets
for boundary outputs; replacing them with fresh nets disconnects their readers.
Connect every output as well as every input before deleting the old group.
Check fresh instance and internal-net names for collisions before mutation.
Temporary multiple drivers during construction must be gone before export.
Keep sequential cells and clock/reset connections outside the selected group.
Formal verification and physical timing measurement remain separate checks.

This example defines only the requested edit. Helpers from other examples are
independent building blocks, not required boilerplate. Include a helper only
when this edit actually needs its behavior, along with its called dependencies.
