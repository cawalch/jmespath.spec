---
title: Ternary Conditional Expression
author: Maxime Labelle
status: accepted
created: 13-05-2025
jep: 21
---

# Ternary Conditional Expression

## Abstract

This JEP introduces a new expression to the JMESPath specification to support a
native **if-then-else** ternary conditional operation.

---

## Motivation

Most programming languages provide a ternary operator for concise if-then-else
logic within a single statement, often using the `?:` syntax:

> `<condition> ? <expression-if-true> : <expression-if-false>`

While JMESPath does not currently have a ternary operator, the community has
relied on workarounds. An expression like `(x || y) && z` is the
[closest approximation](https://github.com/jmespath-community/jmespath.spec/wiki/ternary-operator),
but it suffers from a critical limitation: if `x` is falsey, the expression
incorrectly evaluates to `z` instead of `y`.

More complex workarounds, such as the "array-wrapping" trick
`(x && [y] || [z])[0]`, can produce the correct result but are unintuitive and
not user-friendly, especially for newcomers.

For these reasons, we propose adding a true ternary operator to the language to
provide a clear, standard, and reliable way to perform conditional evaluation.

---

## Specification

### Grammar Changes

The following productions will be added to the JMESPath grammar:

```abnf
expression         =/ ternary-expression
ternary-expression = expression "?" expression ":" expression
```

A `ternary-expression` takes the form
`condition ? true-expression : false-expression`, where `condition`,
`true-expression`, and `false-expression` are all valid JMESPath expressions.

The expression is evaluated as follows:

1.  The `condition` expression is evaluated.
2.  If the result is "truthy" (not a "falsey" value), the `true-expression` is
    evaluated, and its result becomes the result of the entire expression.
3.  Otherwise, the `false-expression` is evaluated, and its result is used.

The following values are considered **falsey** in JMESPath:

- The boolean value `false`
- The value `null`
- An empty string `""`
- An empty object `{}`
- An empty array `[]`

All other values are considered **truthy**.

Ternary expressions are **right-associative**, which allows them to be chained
without parentheses. The expression `a ? b : c ? d : e` is parsed as
`a ? b : (c ? d : e)`.

### Precedence

The ternary conditional operator (`? :`) has a lower precedence than the logical
`||` and `&&` operators but a higher precedence than the pipe `|` operator. The
order of precedence, from lowest to highest, is:

- `|` (Pipe)
- `? :` (Ternary)
- `||` (OR)
- `&&` (AND)

---

## Compliance Tests

A new `ternary.json` test file will be added to the compliance suite to verify
the operator's behavior, including its handling of falsey values and its
precedence rules.

```json
[
  {
    "given": {
      "true_val": true,
      "false_val": false,
      "foo": "foo",
      "bar": "bar",
      "baz": "baz",
      "qux": "qux",
      "quux": "quux"
    },
    "cases": [
      { "expression": "true_val ? foo : bar", "result": "foo" },
      { "expression": "false_val ? foo : bar", "result": "bar" },
      { "expression": "`null` ? foo : bar", "result": "bar" },
      { "expression": "`[]` ? foo : bar", "result": "bar" },
      { "expression": "`{}` ? foo : bar", "result": "bar" },
      { "expression": "'' ? foo : bar", "result": "bar" },
      { "expression": "false_val ? foo | bar | @ : baz", "result": "baz" },
      { "expression": "foo ? bar ? baz : qux : quux", "result": "baz" },
      { "expression": "true_val || false_val ? foo : bar", "result": "foo" },
      { "expression": "true_val && false_val ? foo : bar", "result": "bar" }
    ]
  }
]
```

---

## Alternatives

### Status Quo

While achieving conditional logic is technically possible with existing syntax,
the workarounds are flawed and unintuitive, as noted in the motivation.

### Switch/Case Syntax

A more general "switch-case" syntax was considered. However, the `if-then-else`
pattern is so ubiquitous that a dedicated ternary operator is justified. A more
complex switch syntax is not precluded by this change and could be revisited in
the future.

### `if()` Function

An alternative was to introduce this feature as a function, such as
`if(condition, true_expr, false_expr)`. While this is possible today in
implementations that support custom functions, we believe a native operator is
the most user-friendly and least surprising approach, as it aligns with user
expectations from many other languages.
