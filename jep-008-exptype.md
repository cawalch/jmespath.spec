---
title: Expression Types
jep: 8
author: James Saryerwinnie
status: accepted
created: 2013-03-02
---

# Expression Types

## Abstract

This JEP proposes grammar modifications to JMESPath to support expression
references within functions. These references enable functions like `sort_by`,
`max_by`, and `min_by` that accept expressions as arguments. This allows
operations such as sorting arrays based on dynamically evaluated expressions
applied to each element.

## Motivation

A common requirement in expression languages is sorting JSON objects by specific
keys. Consider this example data:

```
{
  "people": [
       {"age": 20, "age_str": "20", "bool": true, "name": "a", "extra": "foo"},
       {"age": 40, "age_str": "40", "bool": false, "name": "b", "extra": "bar"},
       {"age": 30, "age_str": "30", "bool": true, "name": "c"},
       {"age": 50, "age_str": "50", "bool": false, "name": "d"},
       {"age": 10, "age_str": "10", "bool": true, "name": 3}
  ]
}
```

Currently, JMESPath cannot sort the `people` array by the `age` key.
Additionally, the `sort` function isn't defined for objects, making array
sorting impossible without specifying a sort key.

This limitation extends beyond sorting. The concept can be generalized: instead
of requiring a specific key name, functions should accept expressions evaluated
against each array element. While simplest cases use identifiers (e.g., `age`),
more complex expressions like `foo.bar.baz` should also work.

A naive implementation might define:

```
sort_by(array arg1, expression)
```

Called as:

```
sort_by(people, age)
sort_by(people, to_number(age_str))
```

However, this approach fails due to standard argument resolution:

```
sort_by(people, age)

# 1. Resolve people
arg1 = search(people, <input data>) → [{"age": ...}, {...}]

# 2. Resolve age
arg2 = search(age, <input data>) → null

sort_by([{"age": ...}, {...}], null)
```

The second argument evaluates against the current node, and `age` resolves to
`null` since the root object lacks an `age` key. We need a mechanism to specify
that an argument should represent an unevaluated expression:

```
arg = search(<some expression>, <input data>) → <expression: age>
```

The correct function definition would then be:

```
sort_by(array arg1, expression arg2)
```

## Specification

The grammar rule for function arguments will be updated to:

```
function-arg        = expression /
                      current-node /
                      "&" expression
```

Evaluating an expression reference (prefixed with `&`) returns an object of type
"expression" rather than evaluating the expression immediately. JMESPath data
types now include:

- number (integers and double-precision floating-point values)
- string
- boolean (`true` or `false`)
- array (ordered sequence of values)
- object (unordered collection of key-value pairs)
- null
- expression (denoted by `&expression`)

Function signatures can specify the `expression` type. Similar to how
`array[type]` specifies array element types, `expression->type` specifies the
expected return type of the expression.

Any valid expression can follow the `&` prefix:

```
sort_by(people, &foo.bar.baz)
sort_by(people, &foo.bar[0].baz)
sort_by(people, &to_number(foo[0].bar))
```

### Additional Functions

#### sort_by

```
sort_by(array elements, expression->number|expression->string expr)
```

Sorts an array using `expr` as the sort key. Follows the same sorting logic as
the `sort` function.

| Expression                                | Result                                                |
| ----------------------------------------- | ----------------------------------------------------- |
| `sort_by(people, &age)[].age`             | [10, 20, 30, 40, 50]                                  |
| `sort_by(people, &age)[0]`                | {"age": 10, "age_str": "10", "bool": true, "name": 3} |
| `sort_by(people, &to_number(age_str))[0]` | {"age": 10, "age_str": "10", "bool": true, "name": 3} |

#### max_by

```
max_by(array elements, expression->number expr)
```

Returns the maximum element in an array using `expr` as the comparison key. The
entire element is returned.

| Expression                            | Result                                                   |
| ------------------------------------- | -------------------------------------------------------- |
| `max_by(people, &age)`                | {"age": 50, "age_str": "50", "bool": false, "name": "d"} |
| `max_by(people, &age).age`            | 50                                                       |
| `max_by(people, &to_number(age_str))` | {"age": 50, "age_str": "50", "bool": false, "name": "d"} |
| `max_by(people, &age_str)`            | <error: invalid-type>                                    |
| `max_by(people, age)`                 | <error: invalid-type>                                    |

#### min_by

```
min_by(array elements, expression->number expr)
```

Returns the minimum element in an array using `expr` as the comparison key.

| Expression                            | Result                                                |
| ------------------------------------- | ----------------------------------------------------- |
| `min_by(people, &age)`                | {"age": 10, "age_str": "10", "bool": true, "name": 3} |
| `min_by(people, &age).age`            | 10                                                    |
| `min_by(people, &to_number(age_str))` | {"age": 10, "age_str": "10", "bool": true, "name": 3} |
| `min_by(people, &age_str)`            | <error: invalid-type>                                 |
| `min_by(people, age)`                 | <error: invalid-type>                                 |

## Alternatives

Several alternative approaches were considered:

#### Logic in Argument Resolver

An earlier proposal (originally in JEP-3 but later removed) suggested no
syntactic construct for expression references. Instead, function signatures
would dictate argument resolution:

```
sort_by(array arg1, any arg2)
arg1 → resolved
arg2 → not resolved
```

The argument resolver would inspect function specifications:

```
call-function(current-data)
arglist = []
for each argspec in functions-argspec:
    if argspec.should_resolve:
      arglist ← resolve(argument, current-data)
    else
      arglist ← argument
type-check(arglist)
return invoke-function(arglist)
```

This approach was rejected for several reasons:

- It imposes specific implementation constraints
- It would be challenging to implement in a bytecode VM, where CALL instructions
  typically resolve arguments onto the stack before function invocation
- It deviates from standard function implementation models

#### Specifying Expressions as Strings

Another alternative proposed using string literals for expressions:

```
sort_by(people, `age`)
sort_by(people, `foo.bar.baz`)
```

This approach was rejected because:

- It complicates implementations, requiring AST nodes or visitors to access the
  parser
- It converts potential compile-time errors into runtime errors, as expression
  validation occurs during function execution rather than during compilation
