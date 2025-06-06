---
title: Functions
jep: 3a
author: Michael Dowling, James Saryerwinnie, Maxime Labelle
created: 27-Nov-2013
semver: MINOR
status: accepted
---

# Functions

## Addendum

| Date          | Description                                 |
| ------------- | ------------------------------------------- |
| 16-March-2023 | Clarified string data type and containment. |

## Abstract

This document proposes modifying the
[JMESPath grammar](https://jmespath.org/specification.html#grammar) to support
function expressions.

**Note**: this document uses
[JEP-12 JSON Literals](https://github.com/jmespath-community/jmespath.spec/blob/main/jep-012-raw-string-literals.md#abnf)
such as strings like `` `"foo"` `` and numbers like `` `-1` ``.

## Motivation

Functions allow users to easily transform and filter data in JMESPath
expressions. As JMESPath is currently implemented, functions would be very
useful in `multi-select-list` and `multi-select-hash` expressions to format the
output of an expression to contain data that might not have been in the original
JSON input. Combined with filtered expressions, functions would be a powerful
mechanism to perform any kind of special comparisons for things like `length()`,
`contains()`, etc.

## Data Types

In order to support functions, a type system is needed. The JSON types are used:

- number (integers and double-precision floating-point format in JSON)

- string (a sequence of Unicode
  [code points](https://unicode.org/glossary/#code_point). Note that a code
  point is distinct from a [code unit](https://unicode.org/glossary/#code_unit))

- boolean (`true` or `false`)

- array (an ordered, sequence of values)

- object (an unordered collection of key value pairs)

- null

## Syntax Changes

Functions are defined in the `function-expression` rule below. A function
expression is an `expression` itself, and is valid any place an `expression` is
allowed.

The grammar will require the following grammar additions:

```abnf
function-expression = unquoted-string  (
                        no-args  /
                        one-or-more-args )
no-args             = "(" ")"
one-or-more-args    = "(" ( function-arg *( "," function-arg ) ) ")"
function-arg        = expression / number / current-node
current-node        = "@"
```

`expression` will need to be updated to add the `function-expression`
production:

```abnf
expression        = sub-expression / index-expression / or-expression / identifier / "*"
expression        =/ multi-select-list / multi-select-hash
expression        =/ literal / function-expression
```

A function can accept any number of arguments, and each argument can be an
expression. Each function must define a signature that specifies the number and
allowed types of its expected arguments. Functions can be variadic.

### current-node

The `current-node` token can be used to represent the current node being
evaluated. The `current-node` token is useful for functions that require the
current node being evaluated as an argument. For example, the following
expression creates an array containing the total number of elements in the `foo`
object followed by the value of `foo.bar`.

```jmespath
foo[].[count(@), bar]
```

JMESPath assumes that all function arguments operate on the current node unless
the argument is a `literal` or `number` token. Because of this, an expression
such as `@.bar` would be equivalent to just `bar`, so the current node is only
allowed as a bare expression.

#### current-node state

At the start of an expression, the value of the current node is the data being
evaluated by the JMESPath expression. As an expression is evaluated, the value
the current node represents MUST change to reflect the node currently being
evaluated. When in a projection, the current node value MUST be changed to the
node currently being evaluated by the projection.

## Function Evaluation

Functions are evaluated in applicative order. Each argument must be an
expression, each argument expression must be evaluated before evaluating the
function. The function is then called with the evaluated function arguments. The
result of the `function-expression` is the result returned by the function call.
If a `function-expression` is evaluated for a function that does not exist, the
JMESPath implementation must indicate to the caller that an `unknown-function`
error has occurred. How and when this error is raised is implementation
specific, but implementations should indicate to the caller that this specific
error occurred.

Functions can either have a specific arity or be variadic with a minimum number
of arguments. If a `function-expression` is encountered where the arity does not
match or the minimum number of arguments for a variadic function is not
provided, then implementations must indicate to the caller that an
`invalid-arity` error occurred. How and when this error is raised is
implementation specific.

Each function signature declares the types of its input parameters. If any type
constraints are not met, implementations must indicate that an `invalid-type`
error occurred.

In order to accommodate type constraints, functions are provided to convert
types to other types (`to_string`, `to_number`) which are defined below. No
explicit type conversion happens unless a user specifically uses one of these
type conversion functions.

Function expressions are allowed within sub expressions. This allows functions
to be used with projections, which can enable functions to be applied to every
element in a projection. For example, given the input data of
`["1", "2", "3", "notanumber", true]`, the following expression can be used to
convert (and filter) all elements to numbers:

```jmespath
search([].to_number(@), ["1", "2", "3", "notanumber", true]) -> [1, 2, 3]
```

This provides a simple mechanism to explicitly convert types when needed.

## Built-in Functions

JMESPath has various built-in functions that operate on different data types,
documented below. Each function below has a signature that defines the expected
types of the input and the type of the returned output:

```abnf
return_type function_name(type $argname)
return_type function_name2(type1|type2 $argname)
```

If a function can accept multiple types for an input value, then the multiple
types are separated with `|`. If the resolved arguments do not match the types
specified in the signature, an `invalid-type` error occurs.

The `array` type can further specify requirements on the type of the elements to
enforce homogeneous types. The subtype is surrounded by `[type]`, for example,
the function signature below requires its input argument resolves to an array of
numbers:

```abnf
return_type foo(array[number] $argname)
```

As a shorthand, the type `any` is used to indicate that the argument can be of
any type (`array|object|number|string|boolean|null`).

The first function below, `abs` is discussed in detail to demonstrate the above
points. Subsequent function definitions will not include these details for
brevity, but the same rules apply.

**NOTE**: All string related functions are defined on the basis of Unicode code
points; they do not take normalization into account.

### abs

```abnf
number abs(number $value)
```

Returns the absolute value of the provided argument. The signature indicates
that a number is returned, and that the input argument `$value` **must** resolve
to a number, otherwise an `invalid-type` error is triggered.

Below is a worked example. Given:

```json
{ "foo": -1, "bar": "2" }
```

Evaluating `abs(foo)` works as follows:

1. Evaluate the input argument against the current data:

   ```abnf
   search(foo, {"foo": -1, "bar": 2"}) -> -1
   ```

2. Validate the type of the resolved argument. In this case `-1` is of type
   `number` so it passes the type check.

3. Call the function with the resolved argument:

   ```abnf
   abs(-1) -> 1
   ```

4. The value of `1` is the resolved value of the function expression `abs(foo)`.

Below is the same steps for evaluating `abs(bar)`:

1. Evaluate the input argument against the current data:

   ```abnf
   search(bar, {"foo": -1, "bar": 2"}) -> "2"
   ```

2. Validate the type of the resolved argument. In this case `"2"` is of type
   `string` so immediately indicate that an `invalid-type` error occurred.

As a final example, here is the steps for evaluating `abs(to_number(bar))`:

1. Evaluate the input argument against the current data:

   ```abnf
   search(to_number(bar), {"foo": -1, "bar": "2"})
   ```

   This requires following the same process for `to_number(bar)`:

   1. Evaluate the input argument against the current data:

      ```abnf
      search(bar, {"foo": -1, "bar": "2"}) -> "2"
      ```

   2. Validate the type of the resolved argument. In this case `"2"` is of type
      `string` so it passes the type check.
   3. Call the function with the resolved argument:

      ```abnf
      to_number("2") -> 2
      ```

   4. The value of `2` is the resolved value of the function expression
      `to_number(bar)`.

2. Validate the type of the resolved argument. In this case `2` is of type
   `number` so it passes the type check.

3. Call the function with the final resolved argument:

   ```abnf
   abs(2) -> 2
   ```

4. The value of `2` is the resolved value of the function expression
   `abs(to_number(bar))`.

| Expression         | Result                  |
| ------------------ | ----------------------- |
| `` abs(`1`) ``     | `1`                     |
| `` abs(`-1`) ``    | `1`                     |
| `` abs(`"abc"`) `` | `<error: invalid-type>` |

### avg

```abnf
number avg(array[number] $elements)
```

Returns the average of the elements in the provided array.

An empty array will produce a return value of null.

| Given             | Expression | Result                  |
| ----------------- | ---------- | ----------------------- |
| `[10, 15, 20]`    | `avg(@)`   | 15                      |
| `[10, false, 20]` | `avg(@)`   | `<error: invalid-type>` |
| `[false]`         | `avg(@)`   | `<error: invalid-type>` |
| `false`           | `avg(@)`   | `<error: invalid-type>` |

### ceil

```abnf
number ceil(number $value)
```

Returns the next highest integer value by rounding up if necessary.

| Expression          | Result                  |
| ------------------- | ----------------------- |
| `` ceil(`1.001`) `` | 2                       |
| `` ceil(`1.9`) ``   | 2                       |
| `` ceil(`1`) ``     | 1                       |
| `` ceil(`"abc"`) `` | `<error: invalid-type>` |

### contains

```abnf
boolean contains(array|string $subject, any $search)
```

Returns `true` if the given `$subject` contains the provided `$search` value.

If `$subject` is an array, this function returns true if one of the elements in
the array is equal to the provided `$search` value.

If the provided `$subject` is a string, this function returns true if there
exists within the `$subject` string at least one occurrence of the `$search`
string. If the `$search` value is not a string, the function MUST return
`false`.

| Given        | Expression                          | Result                  |
| ------------ | ----------------------------------- | ----------------------- |
| n/a          | `` contains(`"foobar"`, `"foo"`) `` | `true`                  |
| n/a          | `` contains(`"foobar"`, `"not"`) `` | `false`                 |
| n/a          | `` contains(`"foobar"`, `"bar"`) `` | `true`                  |
| n/a          | `` contains(`false`, `"bar"`) ``    | `<error: invalid-type>` |
| n/a          | `` contains(`"foobar"`, `123`) ``   | `false`                 |
| `["a", "b"]` | `` contains(@, `"a"`) ``            | `true`                  |
| `["a"]`      | `` contains(@, `"a"`) ``            | `true`                  |
| `["a"]`      | `` contains(@, `"b"`) ``            | `false`                 |

### floor

```abnf
number floor(number $value)
```

Returns the next lowest integer value by rounding down if necessary.

| Expression           | Result |
| -------------------- | ------ |
| `` floor(`1.001`) `` | 1      |
| `` floor(`1.9`) ``   | 1      |
| `` floor(`1`) ``     | 1      |

### join

```abnf
string join(string $glue, array[string] $stringsarray)
```

Returns all of the elements from the provided `$stringsarray` array joined
together using the `$glue` argument as a separator between each.

| Given               | Expression            | Result                  |
| ------------------- | --------------------- | ----------------------- |
| `["a", "b"]`        | `` join(`", "`, @) `` | "a, b"                  |
| `["a", "b"]`        | `` join(`""`, @) ``   | "ab"                    |
| `["a", false, "b"]` | `` join(`", "`, @) `` | `<error: invalid-type>` |
| `[false]`           | `` join(`", "`, @) `` | `<error: invalid-type>` |

### keys

```abnf
array keys(object $obj)
```

Returns an array containing the keys of the provided object.

| Given                          | Expression | Result                  |
| ------------------------------ | ---------- | ----------------------- |
| `{"foo": "baz", "bar": "bam"}` | `keys(@)`  | `["foo", "bar"]`        |
| `{}`                           | `keys(@)`  | `[]`                    |
| `false`                        | `keys(@)`  | `<error: invalid-type>` |
| `[b, a, c]`                    | `keys(@)`  | `<error: invalid-type>` |

### length

```abnf
number length(string|array|object $subject)
```

Returns the length of the given argument using the following types rules:

1. string: returns the number of code points in the string

2. array: returns the number of elements in the array

3. object: returns the number of key-value pairs in the object

| Given                          | Expression            | Result                  |
| ------------------------------ | --------------------- | ----------------------- |
| n/a                            | `` length(`"abc"`) `` | 3                       |
| "current"                      | `length(@)`           | 7                       |
| "current"                      | `length(not_there)`   | `<error: invalid-type>` |
| `["a", "b", "c"]`              | `length(@)`           | 3                       |
| `[]`                           | `length(@)`           | 0                       |
| `{}`                           | `length(@)`           | 0                       |
| `{"foo": "bar", "baz": "bam"}` | `length(@)`           | 2                       |

### max

```abnf
number max(array[number] $collection)
```

Returns the highest found number in the provided array argument.

An empty array will produce a return value of null.

| Given             | Expression | Result                  |
| ----------------- | ---------- | ----------------------- |
| `[10, 15]`        | `max(@)`   | 15                      |
| `[10, false, 20]` | `max(@)`   | `<error: invalid-type>` |

### min

```abnf
number min(array[number] $collection)
```

Returns the lowest found number in the provided `$collection` argument.

An empty array will produce a return value of null.

| Given             | Expression | Result                  |
| ----------------- | ---------- | ----------------------- |
| `[10, 15]`        | `min(@)`   | 10                      |
| `[10, false, 20]` | `min(@)`   | `<error: invalid-type>` |

### sort

```abnf
array sort(array $list)
```

This function accepts an array `$list` argument and returns the sorted elements
of the `$list` as an array.

The array must be a list of strings or numbers. Sorting strings is based on code
points. Locale is not taken into account.

| Given               | Expression | Result              |
| ------------------- | ---------- | ------------------- |
| `[b, a, c]`         | `sort(@)`  | `[a, b, c]`         |
| `[1, a, c]`         | `sort(@)`  | `[1, a, c]`         |
| `[false, [], null]` | `sort(@)`  | `[[], null, false]` |
| `[[], {}, false]`   | `sort(@)`  | `[{}, [], false]`   |
| `{"a": 1, "b": 2}`  | `sort(@)`  | `null`              |
| `false`             | `sort(@)`  | `null`              |

### to_string

```abnf
string to_string(string|number|array|object|boolean $arg)
```

- string - Returns the passed in value.

- number/array/object/boolean - The JSON encoded value. The JSON encoder should
  emit the encoded JSON value without adding any additional new lines.

| Given  | Expression           | Result |
| ------ | -------------------- | ------ |
| `null` | `` to_string(`2`) `` | `"2"`  |

### to_number

```abnf
number to_number(string|number $arg)
```

- string - Returns the parsed number. Any string that conforms to the
  `json-number` production is supported.

- number - Returns the passed in value.
- array - null
- object - null
- boolean - null

### type

```abnf
string type(array|object|string|number|boolean|null $subject)
```

Returns the JMESPath type of the given `$subject` argument as a string value.

The return value MUST be one of the following:

- number
- string
- boolean
- array
- object
- null

| Given            | Expression | Result    |
| ---------------- | ---------- | --------- |
| "foo"            | `type(@)`  | "string"  |
| `true`           | `type(@)`  | "boolean" |
| `false`          | `type(@)`  | "boolean" |
| `null`           | `type(@)`  | "null"    |
| 123              | `type(@)`  | number    |
| 123.05           | `type(@)`  | number    |
| `["abc"]`        | `type(@)`  | "array"   |
| `{"abc": "123"}` | `type(@)`  | "object"  |

### values

```abnf
array values(object $obj)
```

Returns the values of the provided object.

| Given                          | Expression  | Result                  |
| ------------------------------ | ----------- | ----------------------- |
| `{"foo": "baz", "bar": "bam"}` | `values(@)` | `["baz", "bam"]`        |
| `["a", "b"]`                   | `values(@)` | `<error: invalid-type>` |
| `false`                        | `values(@)` | `<error: invalid-type>` |

## Compliance Tests

A `functions.json` will be added to the compliance test suite. The test suite
will add the following new error types:

- unknown-function
- invalid-arity
- invalid-type

The compliance does not specify **when** the errors are raised, as this will
depend on implementation details. For an implementation to be compliant it needs
to indicate that an error occurred while attempting to evaluate the JMESPath
expression.

## History

- This JEP originally proposed the literal syntax. The literal portion of this
  JEP was removed and added instead to JEP-7.

- This JEP originally specified that types matches should return null. This has
  been updated to specify that an invalid type error should occur instead.
