---
title: group_by
id: 18
parent: 3a
status: accepted
author: Maxime Labelle
created: 2022-08-05
---

# group_by function

## Abstract

This JEP introduces a new `group_by()` function.

## Specification

### group_by

```
object group_by(array[object] $elements, expression->string $expr)
```

Groups an array `$elements` of objects using an expression `$expr` as the group
key. The `$expr` expression is applied to each element in the array `$elements`,
and the resulting value is used as a group key.

The result is an object whose keys are the string values returned by the
expression and whose respective values are arrays of objects that produced the
corresponding group key.

Objects for which the `$expr` expression evaluates to `null` are excluded from
the output.

If the result of applying the `$expr` expression against the current array
element results in a type other than `string` or `null`, an `invalid-type` error
MUST be raised.

### Examples

Given the following input JSON document:

```json
{
  "items": [
    { "spec": { "nodeName": "node_01", "other": "values_01" } },
    { "spec": { "nodeName": "node_02", "other": "values_02" } },
    { "spec": { "nodeName": "node_03", "other": "values_03" } },
    { "spec": { "nodeName": "node_01", "other": "values_04" } }
  ]
}
```

Example:

| Expression                        | Result                                                                                                                                                                                                                                                                        |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `group_by(items, &spec.nodeName)` | `{"node_01": [{"spec": {"nodeName": "node_01", "other": "values_01"}}, {"spec": {"nodeName": "node_01", "other": "values_04"}}], "node_02": [{"spec": {"nodeName": "node_02", "other": "values_02"}}], "node_03": [{"spec": {"nodeName": "node_03", "other": "values_03"}}]}` |

Given the following input JSON document:

```json
{
  "array": [
    { "name": "one", "b": true },
    { "name": "two", "b": false },
    { "b": false }
  ]
}
```

Here are additional examples:

| Expression               | Result                                                               |
| ------------------------ | -------------------------------------------------------------------- |
| `group_by(array, &name)` | `{"one":[{"name":"one","b":true}],"two":[{"name":"two","b":false}]}` |
| `group_by(array, &b)`    | `invalid-type`                                                       |
