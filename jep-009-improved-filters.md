---
title: Improved Filters
jep: 9
status: accepted
created: 2014-07-07
author: James Saryerwinnie
---

# Improved Filters

## Abstract

JEP-7 introduced filter expressions, enabling list elements to be selected based
on matching expressions against each element. While this concept proved useful,
the comparator expressions lacked sufficient capability to accommodate numerous
common query patterns. This JEP expands filter expressions by introducing
support for `and-expressions`, `not-expressions`, `paren-expressions`, and
`unary-expressions`. These additions significantly enhance filter expression
capabilities, enabling sufficiently powerful queries to handle the majority of
practical use cases.

## Motivation

JEP-7 introduced filter queries with the following structure:

```
foo[?lhs comparator rhs]
```

where the left-hand side (lhs) and right-hand side (rhs) are both `expression`s,
and `comparator` is one of `==`, `!=`, `<`, `<=`, `>`, or `>=`.

This added a valuable feature to JMESPath: the ability to filter lists by
evaluating expressions against each element.

Since JEP-7's incorporation into JMESPath, several limitations have emerged
where filter expressions cannot express required queries. Below are examples of
missing features.

### Or Expressions

Users need the ability to filter based on matching one or more expressions. For
example, given:

```
{
  "cities": [
    {"name": "Seattle", "state": "WA"},
    {"name": "Los Angeles", "state": "CA"},
    {"name": "Bellevue", "state": "WA"},
    {"name": "New York", "state": "NY"},
    {"name": "San Antonio", "state": "TX"},
    {"name": "Portland", "state": "OR"}
  ]
}
```

A user might want to select locations on the west coast (cities in `WA`, `OR`,
or `CA`). This cannot be expressed with the current
`expression comparator expression` grammar. Ideally, users should be able to
write:

```
cities[?state == `WA` || state == `OR` || state == `CA`]
```

While JMESPath already supports OR expressions, they weren't permitted within
filter expressions.

### And Expressions

Filter expressions lack support for AND expressions. It's somewhat inconsistent
that JMESPath supports OR expressions but not AND expressions. For example,
given user accounts with permissions:

```
{
  "users": [
    {"name": "user1", "type": "normal", "allowed_hosts": ["a", "b"]},
    {"name": "user2", "type": "admin", "allowed_hosts": ["a", "b"]},
    {"name": "user3", "type": "normal", "allowed_hosts": ["c", "d"]},
    {"name": "user4", "type": "admin", "allowed_hosts": ["c", "d"]},
    {"name": "user5", "type": "normal", "allowed_hosts": ["c", "d"]},
    {"name": "user6", "type": "normal", "allowed_hosts": ["c", "d"]}
  ]
}
```

We'd like to find admin users with access to host `c`. The ideal filter
expression would be:

```
users[?type == `admin` && contains(allowed_hosts, `c`)]
```

### Unary Expressions

Consider if statements in languages like C or Java. While you can write:

```
if (foo == bar) { ... }
```

You can also use unary expressions:

```
if (allowed_access) { ... }
```

or:

```
if (!allowed_access) { ... }
```

Adding unary expression support creates natural syntax when filtering against
boolean values. Instead of:

```
foo[?boolean_var == `true`]
```

Users could write:

```
foo[?boolean_var]
```

For a more realistic example, using modified `users` data:

```
{
  "users": [
    {"name": "user1", "is_admin": false, "disabled": false},
    {"name": "user2", "is_admin": true, "disabled": true},
    {"name": "user3", "is_admin": false, "disabled": false},
    {"name": "user4", "is_admin": true, "disabled": false},
    {"name": "user5", "is_admin": false, "disabled": true},
    {"name": "user6", "is_admin": false, "disabled": false}
  ]
}
```

To get names of all enabled admin users, we could write:

```
users[?is_admin == `true` && disabled == `false`]
```

But it's more natural and concise to write:

```
users[?is_admin && !disabled]
```

While not strictly necessary, the primary reason for adding unary expression
support is user expectation. Users anticipate this syntax and are surprised when
it's unavailable. Especially now that we're adopting C-like syntax for
filtering, users will increasingly expect unary expressions.

### Paren Expressions

Once `||` and `&&` operators are introduced, there will be cases where
overriding operator precedence is necessary.

A `paren-expression` allows users to override expression precedence, e.g.,
`(a || b) && c` instead of the default `a || (b && c)` for `a || b && c`.

## Specification

Several grammar updates are required:

```
and-expression         = expression "&&" expression
not-expression         = "!" expression
paren-expression       = "(" expression ")"
```

Additionally, the `bracket-specifier` rule becomes more general:

```
bracket-specifier      =/ "[?" expression "]"
```

The `list-filter-expression` is now a more general `comparator-expression`:

```
comparator-expression  = expression comparator expression
```

Which integrates into the expression hierarchy:

```
expression            /= comparator-expression
```

And the `current-node` becomes a valid general expression:

```
expression            /= current-node
```

### Operator Precedence

This JEP introduces AND expressions, which would typically be defined as:

```
expression     = or-expression / and-expression / not-expression
or-expression  = expression "||" expression
and-expression = expression "&&" expression
not-expression = "!" expression
```

However, this structure makes correct precedence parsing impossible. A more
standard approach is:

```
expression          = or-expression
or-expression       = and-expression "||" and-expression
and-expression      = not-expression "&&" not-expression
not-expression      = "!" expression
```

The precedence for new boolean expressions matches most programming languages,
ordered from weakest to tightest binding:

- OR - `||`
- AND - `&&`
- Unary NOT - `!`

Thus, `a || b && c` parses as `a || (b && c)` rather than `(a || b) && c`.

The complete operator precedence list now reads:

- Pipe - `|`
- OR - `||`
- AND - `&&`
- Unary NOT - `!`
- Rbracket - `]`

Since these expressions are now valid general `expressions`, their semantics
outside original contexts must be defined.

### And Expressions

JMESPath defines the following "false-like" values:

- Empty list: `[]`
- Empty object: `{} `
- Empty string: `""`
- False boolean: `false`
- Null value: `null`

Any value not in this list is "truth-like."

An `and-expression` follows standard semantics:

- If left-hand side is truth-like, return right-hand side value
- Otherwise, return left-hand side value

This produces the expected truth table:

| LHS   | RHS   | Result |
| ----- | ----- | ------ |
| True  | True  | True   |
| True  | False | False  |
| False | True  | False  |
| False | False | False  |

This matches the standard truth table for
[logical conjunction (AND)](https://en.wikipedia.org/wiki/Truth_table#Logical_conjunction_.28AND.29).

#### Examples

```
search(`true && false`, {"true": true, "false": false}) → false
search(`Number && EmptyList`, {"Number": 5, "EmptyList": []}) → []
search(`foo[?a == `1` && b == `2`], {"foo": [{"a": 1, "b": 2}, {"a": 1, "b": 3}]}) → [{"a": 1, "b": 2}]
```

### Not Expressions

A `not-expression` negates an expression's result:

- If expression yields truth-like value, result becomes `false`
- If expression yields false-like value, result becomes `true`

#### Examples

```
search(`!true`, {"true": true}) → false
search(`!false`, {"false": false}) → true
search(`!Number`, {"Number": 5}) → false
search(`!EmptyList`, {"EmptyList": []}) → true
```

### Paren Expressions

A `paren-expression` allows overriding expression precedence, e.g.,
`(a || b) && c`.

#### Examples

```
search(`foo[?(a == `1` || b == `2`) && c == `5`], {"foo": [{"a": 1, "b": 2, "c": 3}, {"a": 3, "b": 4}]}) → []
```

## Rationale

This JEP promotes several tokens from specific constructs to the general
`expression` rule:

- The `current-node` (`@`) was previously only allowed in function expressions
  but is now a general `expression`
- The `filter-expression` now accepts any arbitrary `expression`
- The `list-filter-expression` is now a generic `comparator-expression`, which
  is itself a general `expression`

Previous grammar rules were minimally scoped for good reason. As stated in
JEP-7, the specification was "purposefully minimal." In fact, JEP-7 concluded
that "there are several extensions that can be added in future." This JEP
directly implements those recommended extensions from JEP-7, enhancing the
language while maintaining backward compatibility.
