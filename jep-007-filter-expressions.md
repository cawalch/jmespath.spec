---
title: Filter Expressions
jep: 7
author: James Saryerwinnie
status: accepted
created: 2013-12-16
---

# Filter Expressions

## Abstract

This JEP proposes grammar modifications to JMESPath to enable filter
expressions. A filter expression allows list elements to be selected based on
matching expressions. A literal expression is also introduced (from JEP-3) to
enable matching elements against literal values.

## Motivation

A common requirement when querying JSON objects is selecting elements based on
specific values. For example, given this JSON:

```
{"foo": [{"state": "WA", "value": 1},
         {"state": "WA", "value": 2},
         {"state": "CA", "value": 3},
         {"state": "CA", "value": 4}]}
```

A user may want to select all objects in the `foo` list where the `state` key
equals `WA`. This capability doesn't currently exist in JMESPath. This JEP
introduces syntax to address this:

```
foo[?state == `WA`]
```

Additionally, users often need to project expressions onto filtered values. For
instance, selecting the `value` key from all objects with `state` equal to `WA`:

```
foo[?state == `WA`].value
```

would return `[1, 2]`.

## Specification

The updated grammar for filter expressions:

```
bracket-specifier      = "[" (number / "*") "]" / "[]"
bracket-specifier      =/ "[?" list-filter-expression "]"
list-filter-expression = expression comparator expression
comparator             = "<" / "<=" / "==" / ">=" / ">" / "!="
expression             =/ literal
literal                = "`" json-value "`"
literal                =/ "`" 1*(unescaped-literal / escaped-literal) "`"
unescaped-literal      = %x20-21 /       ; space !
                            %x23-5A /   ; # - [
                            %x5D-5F /   ; ] ^ _
                            %x61-7A     ; a-z
                            %x7C-10FFFF ; |}~ ...
escaped-literal        = escaped-char / (escape %x60)
```

The `json-value` rule represents any valid JSON value. While implementations are
recommended to use an existing JSON parser for `json-value`, the grammar is
included below for completeness:

```
json-value = "false" / "null" / "true" / json-object / json-array /
             json-number / json-quoted-string
json-quoted-string = %x22 1*(unescaped-literal / escaped-literal) %x22
begin-array     = ws %x5B ws  ; [ left square bracket
begin-object    = ws %x7B ws  ; { left curly bracket
end-array       = ws %x5D ws  ; ] right square bracket
end-object      = ws %x7D ws  ; } right curly bracket
name-separator  = ws %x3A ws  ; : colon
value-separator = ws %x2C ws  ; , comma
ws              = *(%x20 /              ; Space
                    %x09 /              ; Horizontal tab
                    %x0A /              ; Line feed or New line
                    %x0D                ; Carriage return
                   )
json-object = begin-object [ member *( value-separator member ) ] end-object
member = quoted-string name-separator json-value
json-array = begin-array [ json-value *( value-separator json-value ) ] end-array
json-number = [ minus ] int [ frac ] [ exp ]
decimal-point = %x2E       ; .
digit1-9 = %x31-39         ; 1-9
e = %x65 / %x45            ; e E
exp = e [ minus / plus ] 1*DIGIT
frac = decimal-point 1*DIGIT
int = zero / ( digit1-9 *DIGIT )
minus = %x2D               ; -
plus = %x2B                ; +
zero = %x30                ; 0
```

### Comparison Operators

Supported operations:

- `==`, tests for equality
- `!=`, tests for inequality
- `<`, less than
- `<=`, less than or equal to
- `>`, greater than
- `>=`, greater than or equal to

Operator behavior depends on the evaluated expression types.

#### Equality Operators

For `string`, `number`, `true`, `false`, and `null` types, equality requires an
exact match:

- Two `string` values are equal if their code point sequences are identical
- Literal values (`true`/`false`/`null`) only equal their own literal values
- Two JSON objects are equal if they share identical key sets with equal values
  for each key
- Two JSON arrays are equal if they contain equal elements in the same order
  (for arrays `x` and `y`, `x[i] == y[i]` for all `i`)

#### Ordering Operators

Ordering operators (`>`, `>=`, `<`, `<=`) **only** operate on numbers.
Evaluating other types yields `null`, excluding the element from results. For
example:

```
search('foo[?a<b]', {"foo": [{"a": "char", "b": "char"},
                             {"a": 2, "b": 1},
                             {"a": 1, "b": 2}]})
```

- First element: `"char" < "char"` → invalid type → `null` → excluded
- Second element: `2 < 1` → `false` → excluded
- Third element: `1 < 2` → `true` → included  
  Final result: `[{"a": 1, "b": 2}]`

### Filtering Semantics

When a filter expression matches, the entire element is included in results.
Using the earlier data:

```
{"foo": [{"state": "WA", "value": 1},
         {"state": "WA", "value": 2},
         {"state": "CA", "value": 3},
         {"state": "CA", "value": 4}]}
```

The expression `foo[?state == ```WA```]` returns:

```
[{"state": "WA", "value": 1}, {"state": "WA", "value": 2}]
```

### Literal Expressions

Literal expressions consist of JSON values surrounded by backticks. The backtick
character must be escaped as ```\```` within literals. A two-pass lexer approach
should:

1. Process escaped backticks (``\```` → ` ` ``)
2. Pass the resulting string to a JSON parser

String literals commonly omit surrounding double quotes:

```
`foobar`   → "foobar"
`"foobar"` → "foobar"
`123`      → 123
`"123"`    → "123"
`123.foo`  → "123.foo"
`true`     → true
`"true"`   → "true"
`truee`    → "truee"
```

Literal expressions are prohibited on the right-hand side of subexpressions:

```
foo[*].`literal`  // Invalid
```

But allowed on the left-hand side:

```
`{"foo": "bar"}`.foo
```

They may also appear outside filter expressions, such as in multi-select hashes:

```
{value: foo.bar, type: `multi-select-hash`}
```

## Rationale

The proposed filter syntax balances expressive power with minimalism. Alternate
approaches considered:

- Simple syntax like `foo[bar == baz]` has critical limitations:
  - Cannot filter based on two expressions (e.g., `foo` key equals `bar` key)
  - Literal-on-right syntax increases error risk (`foo[baz == bar]` vs
    `foo[bar == baz]`)
  - Ambiguity with existing multiselect syntax (`[foo]` could mean filter or
    multiselect)
  - Poor readability for complex literals (`[foo == [2]]`)
  - Literal expressions gain utility beyond filters (e.g., multi-select hashes)

This JEP remains intentionally minimal. Future extensions could include:

- Arbitrary expressions within filters (enabling `OR` conditions)  
  Requires defining truthy/falsy values and expression branching semantics.
  Backward-compatible, so deferred.
- Top-level filter expressions  
  Might return boolean masks for list filtering. Requires complementary
  operators for practical use.
