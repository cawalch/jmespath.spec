---
title: Arithmetic Expressions
author: Maxime Labelle
created: 01-August-2022
semver: MINOR
status: accepted
jep: 16
---

# Arithmetic Expressions

## Abstract

This JEP proposes a new `arithmetic-expression` rule in the grammar to support
simple arithmetic operations.

## Motivation

JMESPath supports querying JSON documents that may return numbers at various
stages, but lacks the capability to perform basic arithmetic operations. The
only current arithmetic support is provided by the `sum()` function.

## Specification

To support arithmetic operations, we introduce the following operators:

- `+` addition operator
- `-` subtraction operator
- `*` multiplication operator
- `/` division operator
- `%` modulo operator
- `//` integer division operator

Proper mathematical operators in Unicode SHOULD also be supported:

- `–` (U+2212 MINUS SIGN)
- `÷` (U+00F7 DIVISION SIGN)
- `×` (U+00D7 MULTIPLY SIGN)

### Syntax changes

Arithmetic operators follow standard mathematical precedence, listed here from
highest to lowest:

- Unary operators: `+` (unary plus), `-`/`–` (unary minus)
- Multiplicative operators: `*`/`×`, `/`/`÷`, `%`, `//`
- Additive operators: `+`, `-`/`–`

Operators of the same precedence level are evaluated left-to-right in the
absence of parentheses. Arithmetic operators have lower precedence than the `.`
dot operator (used in sub-expressions) but higher precedence than comparison
operators.

```abnf
expression =/ arithmetic-expression

arithmetic-expression =/ "+" expression
arithmetic-expression =/ ( "-" / "–" ) expression

arithmetic-expression = expression "%" expression
arithmetic-expression =/ expression ( "*" / "×" ) expression
arithmetic-expression =/ expression "+" expression
arithmetic-expression =/ expression ( "-" / "–" ) expression
arithmetic-expression =/ expression ( "/" / "÷" ) expression
arithmetic-expression = expression "//" expression
```

## Examples

| Expression | Result              |
| ---------- | ------------------- |
| `1 + 2`    | `3.0`               |
| `1 – 2`    | `-1.0`              |
| `2 × 4`    | `8.0`               |
| `2 ÷ 3`    | `0.666666666666667` |
| `10 % 3`   | `1.0`               |
| `10 // 3`  | `3.0`               |
| `-1 − +2`  | `-3.0`              |

Since `arithmetic-expression` is invalid on the right-hand side of a
`sub-expression`, use a `pipe-expression`:

| Given                                  | Expression                        | Result |
| -------------------------------------- | --------------------------------- | ------ |
| `{ "a": { "b": 1 }, "c": { "d": 2 } }` | `{ ab: a.b, cd: c.d } \| ab + cd` | `3.0`  |

## Compliance Tests

An `arithmetic.json` file will be added to the compliance test suite. The test
suite will introduce a new error type:

- `invalid-arithmetic`

This error is raised at runtime for:

- Division by zero
- Arithmetic overflow
- Other undefined arithmetic conditions
