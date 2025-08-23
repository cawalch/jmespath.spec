---
title: Root Reference
author: Maxime Labelle
jep: 17
created: 2022-02-25
status: accepted
---

# Root Reference

## Abstract

This JEP proposes a modification to the JMESPath grammar to introduce a **root
reference operator**, `$`. This operator allows an expression to refer to the
original input JSON document from any scope, enabling more complex queries and
data correlation.

---

## Motivation

When a JMESPath expression is evaluated, the context, or **scope**, changes as
the expression is processed. For example, in the expression `foo.bar`, the `foo`
key is first evaluated against the initial JSON document. The result of that
operation then becomes the new scope against which `bar` is evaluated.

A limitation of this model is that once an expression has drilled down into a
nested structure, there is no way to refer to elements outside of the current
scope. A common request is the ability to reference the top-level input document
from within a deeply nested part of an expression.

Consider the following data:

```json
{
  "first_choice": "WA",
  "states": [
    { "name": "WA", "cities": ["Seattle", "Bellevue", "Olympia"] },
    { "name": "CA", "cities": ["Los Angeles", "San Francisco"] },
    { "name": "NY", "cities": ["New York City", "Albany"] }
  ]
}
```

Suppose we want to retrieve the list of cities for the state that matches the
value of the `first_choice` key. Assuming state names are unique, this is not
currently possible in a dynamic way. We can hard-code the value `"WA"`:

```jmespath
states[?name==`"WA"`].cities
```

However, it is impossible to base this filter on the value of `first_choice`,
which exists at the root level, outside the scope of the filter expression. This
JEP proposes a solution to make such queries possible.

---

## Specification

The JMESPath grammar will be updated to support a new token, `$`, which always
refers to the root of the original input JSON document.

This concept is inspired by similar operators in other query languages, such as
`$` in **[JSONPath](https://goessner.net/articles/JsonPath/)** and the leading
`/` in **[XPath](https://www.w3.org/TR/1999/REC-xpath-19991116)**, both of which
designate the document root.

This JEP introduces the following production to the grammar:

```abnf
root-node      = "$"
```

The `expression` production will be updated to include this new token:

```abnf
expression     =/ root-node
```

### Motivating Example

With this change, the expression from the "Motivation" section can now be
written as:

```jmespath
states[?name==$.first_choice].cities[]
```

When evaluated against the example data, this expression correctly resolves
`$.first_choice` to `"WA"`. The filter then finds the matching state object, and
the final projection extracts and flattens the list of cities, producing the
following result:

```json
["Seattle", "Bellevue", "Olympia"]
```

---

## Rationale

This feature standardizes a common request for querying JSON documents and is
already present in some non-standard JMESPath implementations.

This JEP was considered against several alternatives. Most notably, the
[Lexical Scoping proposal (JEP 11)](https://www.google.com/search?q=./jep-011-let-function.md)
introduces a `let()` function that offers a more general and flexible way to
achieve the same result. For instance, using `let()`, the motivating example
could be written as:

```jmespath
let({root: @}, &states[?name==root.first_choice].cities[])
```

Although the Lexical Scoping proposal covers this use case, the `$` root node
operator provides a significantly more direct and ergonomic syntax for the
common task of correlating data against the root document. The two features are
complementary; `$` serves as a convenient shorthand for a frequent pattern,
while `let()` provides a powerful tool for more complex scenarios.
