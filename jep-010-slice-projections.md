---
title: Slice Projections
author: James Saryerwinnie
jep: 10
status: accepted
created: 08-Feb-2015
---

# Slice Projections

## Abstract

This document proposes modifying the semantics of slice expressions to create
projections, bringing consistency with wildcard, flattening, and filtering
projections in JMESPath.

## Motivation

JEP-5 introduced slice expressions, adding Python-style slice semantics to JSON.
However, slicing currently does not produce a projection, causing expressions
like `myarray[:10].foo.bar` to always return `null`.

To access `foo.bar` for each element in an array slice, users must currently
write `myarray[:10][*].foo.bar`.

This JEP proposes that slice expressions should automatically create
projections, eliminating the need for the intermediate `[*]` syntax.

### Rationale

A reasonable objection to this JEP is that it's unnecessary since users can
already create projections from slices using `[*]`. Unlike other JEPs, this
change doesn't enable previously impossible behaviors.

The primary motivation is consistency across projection types. Currently,
JMESPath supports three array projection types:

- List Projections (`foo[*].bar`)
- Filter Projections (`foo[?a==b].bar`)
- Flatten Projections (`foo[].bar`)

Each follows the same general pattern `foo[<selector>].<child-expr>` with
consistent semantics:

1. The left-hand side evaluates to a list (containing original elements or
   elements from sub-arrays in the case of flattening)
2. The right-hand side expression is evaluated against each element in this
   resulting list

The left-hand side creates the list without manipulating individual elements,
while the right-hand side processes each element.

Slices naturally fit this pattern. Like filter projections (which select
elements matching an expression), slice projections select elements within
specific index ranges. Given their semantic similarity to filter projections,
slices should create projections to maintain consistency across the language.

## Specification

Slice expressions will now create projections, becoming the fourth array
projection type in JMESPath:

- List Projections
- Flatten Projections
- Filter Projections
- Slice Projections

A slice projection is evaluated as follows:

1. The slice expression is evaluated to create a new sub-array
2. The right-hand side expression is evaluated against each element in this
   sub-array

For example, given:

```
{
  "people": [
    {"name": "John", "age": 30},
    {"name": "Jane", "age": 25},
    {"name": "Bob", "age": 40}
  ]
}
```

The expression `people[:2].name` will now return `["John", "Jane"]` instead of
`null`.

This JEP does not modify the JMESPath grammar. The existing slice syntax remains
unchanged:

```
slice-expression = [number] ":" [number] [ ":" [number] ]
```

## Impact

The impact on existing JMESPath usage is minimal:

- Expressions like `foo[:10].bar` currently return `null` but will now return
  values
- The only potential compatibility issue involves expressions like
  `foo[:10][0]`, which under the new projection semantics would return a list
  containing the 0th element from each sublist
- However, such expressions were already problematic under current semantics
  (`foo[:10][0]` is equivalent to `foo[0]`), making them rare in practice
- Any users relying on this pattern can simply replace `foo[:10][0]` with
  `foo[0]`

This change enhances usability without compromising backward compatibility for
properly constructed expressions, while bringing much-needed consistency to
JMESPath's projection system.
