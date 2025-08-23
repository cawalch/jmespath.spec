---
title: Evaluation of Pipe Expressions
author: Maxime Labelle
created: 2022-10-29
jep: 19
status: accepted
---

# Evaluation of Pipe Expressions

## Abstract

This JEP proposes changes to the JMESPath specification to clarify the
evaluation behavior of a `pipe-expression`. It also resolves an ambiguity in how
a `sub-expression` is evaluated when its left-hand side results in `null`.

---

## Motivation

### Sub-Expression Evaluation

The specification for
[`sub-expression`](<https://www.google.com/search?q=%5Bhttps://jmespath.org/specification.html%23subexpressions%5D(https://jmespath.org/specification.html%23subexpressions)>)
currently outlines its evaluation with the following pseudocode:

```
left-evaluation = search(left-expression, data)
result = search(right-expression, left-evaluation)
```

This description is ambiguous because the JMESPath compliance tests expect a
sub-expression to "short-circuit" and return `null` immediately if the left-hand
side evaluates to `null`. Therefore, the actual evaluation logic should be:

```
left-evaluation = search(left-expression, data)
if left-evaluation is `null` then
  result = `null`
else
  result = search(right-expression, left-evaluation)
```

### Pipe Expression Evaluation

Conversely, the intuitive behavior for a `pipe-expression` (`|`) is to pass the
result of the left-hand side—**even if it is `null`**—to the right-hand side for
evaluation. This means a pipe expression should _not_ short-circuit, and its
behavior aligns with the simpler, original pseudocode:

```
left-evaluation = search(left-expression, data)
result = search(right-expression, left-evaluation)
```

This JEP formalizes this distinction.

---

## Specification

The following changes will be made to the JMESPath specification.

### Sub-Expressions

The paragraph on sub-expressions will be updated as follows (changes are in
**bold**):

> A sub-expression is evaluated as follows:
>
> - Evaluate the expression on the left with the current JSON document.
> - **If the result of the left expression is `null`, the result of the
>   sub-expression is `null`.**
> - **Otherwise,** \~\~E\~\~evaluate the expression on the right using the
>   result of the left expression as the new JSON document.
>
> _In pseudocode:_
>
> ```
> left-evaluation = search(left-expression, data)
> if left-evaluation is `null` then
>   result = `null`
> else
>   result = search(right-expression, left-evaluation)
> ```

### Pipe Expressions

The paragraph on pipe expressions will be updated to clarify its unique
behaviors:

> A pipe expression combines two expressions, separated by the `|` character. It
> is similar to a sub-expression with **the following** important distinctions:
>
> 1.  Any expression can be used on the right-hand side. A `sub-expression`
>     restricts the type of expression allowed on its right-hand side.
> 2.  A `pipe-expression` stops projections. If the left expression creates a
>     projection, that projection does not propagate to the right-hand side.
> 3.  **A `pipe-expression` evaluates the right expression unconditionally. A
>     `sub-expression` short-circuits evaluation if the result of the left
>     expression is `null`.**
>
> _In pseudocode:_
>
> ```
> left-evaluation = search(left-expression, data)
> result = search(right-expression, left-evaluation)
> ```

### Example

The following table demonstrates the difference in behavior:

| Expression              | Result   | Explanation                                                  |
| ----------------------- | -------- | ------------------------------------------------------------ |
| `` `null` . foo ``      | `null`   | The sub-expression short-circuits because the LHS is `null`. |
| `` `null` \| type(@) `` | `"null"` | The `null` value is piped to the `type()` function.          |

---

## Compliance

The `pipe.json` compliance test file will be updated with test cases that verify
this specified behavior.
