# Constant dataflow
## Constant propagation dataflow contract
<!-- example-purpose: constant propagation | fixed point | worklist | constant dataflow -->

Compose the [graph snapshot](boolean-graph.md), this pure query, and
[deferred reader edits](constant-reader-edits.md). Generate the worklist loop.
Inputs are ordered opaque signal IDs; facts maps known IDs to binary values.
The result is 0/1 only if every compatible assignment agrees; otherwise None.

```python
def restricted_output_value(table, inputs, facts):
    if table is None or table[0] != len(inputs):
        return None
    unknown = [i for i, signal in enumerate(inputs) if signal not in facts]
    base = sum(facts[signal] << i for i, signal in enumerate(inputs) if signal in facts)
    result = None
    for mask in range(1 << len(unknown)):
        assignment = base
        for position, i in enumerate(unknown):
            assignment |= ((mask >> position) & 1) << i
        value = (table[1 + assignment // 64] >> (assignment % 64)) & 1
        if result is None:
            result = value
        elif result != value:
            return None
    return result
```
