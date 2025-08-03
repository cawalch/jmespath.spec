---
title: Array Slice Expressions
jep: 5
author: Michael Dowling
status: accepted
type: standards-track
created: 2013-12-08
---

# Array Slice Expressions

## Abstract

This document proposes modifying the JMESPath grammar to support array slicing
for accessing specific portions of an array.

## Motivation

The current JMESPath grammar does not allow extracting specific portions of an
array.

Array slicing notation provides the following capabilities using an optional
start position, optional stop position, and optional step that can be positive
or negative:

1. Extracting every N indices (e.g., only even `[::2]`, only odd `[1::2]`, etc.)
2. Extracting only elements after a given start position: `[2:]`
3. Extracting only elements before a given stop position: `[:5]`
4. Extracting elements between given start and end positions: `[2:5]`
5. Only the last 5 elements: `[-5:]`
6. The last five elements in reverse order: `[:-5:-1]`
7. Reversing the order of an array: `[::-1]`

## Syntax

This syntax introduces Python-style array slicing that allows a start position,
stop position, and step. This syntax follows the same semantics as Python
slices.

```
[start:stop:step]
```

Each part of the expression is optional. You can omit the start position, stop
position, or step. No more than three values can be provided in a slice
expression.

The step value determines how many indices to skip after each element is
extracted from the array. A step of 1 (the default step) will not skip any
indices. A step value of 2 will skip every other index while extracting values
from an array. A step value of -1 will extract values in reverse order from the
array. A step value of -2 will extract values in reverse order from the array
while skipping every other index.

## Slice Expression Rules

Slice expressions adhere to the following rules:

1. If a negative start position is given, it is calculated as the total length
   of the array plus the given start position.

2. If no start position is given, it is assumed to be 0 if the given step is
   greater than 0 or the end of the array if the given step is less than 0.

3. If a negative stop position is given, it is calculated as the total length of
   the array plus the given stop position.

4. If no stop position is given, it is assumed to be the length of the array if
   the given step is greater than 0 or 0 if the given step is less than 0.

5. If the given step is omitted, it is assumed to be 1.

6. If the given step is 0, an `invalid-value` error must be raised.

7. If the element being sliced is not an array, the result must be `null`.

8. If the element being sliced is an array and yields no results, the result
   must be an empty array.

## Modified Grammar

The following modified JMESPath grammar supports array slicing:

```abnf
expression        = sub-expression / index-expression / or-expression / identifier / "*"
expression        =/ multi-select-list / multi-select-hash
sub-expression    = expression "." expression
or-expression     = expression "||" expression
index-expression  = expression bracket-specifier / bracket-specifier
multi-select-list = "[" ( expression *( "," expression ) ) "]"
multi-select-hash = "{" ( keyval-expr *( "," keyval-expr ) ) "}"
keyval-expr       = identifier ":" expression
bracket-specifier = "[" (number / "*" / slice-expression) "]" / "[]"
slice-expression  = ":"
slice-expression  =/ number ":" number ":" number
slice-expression  =/ number ":"
slice-expression  =/ number ":" ":" number
slice-expression  =/ ":" number
slice-expression  =/ ":" number ":" number
slice-expression  =/ ":" ":" number
number            = ["-"] 1*digit
digit             = "0" / "1" / "2" / "3" / "4" / "5" / "6" / "7" / "8" / "9"
identifier        = 1*char
identifier        =/ quote 1*(unescaped-char / escaped-quote) quote
escaped-quote     = escape quote
unescaped-char    = %x30-10FFFF
escape            = %x5C   ; Back slash: \
quote             = %x22   ; Double quote: "
char              = %x30-39 / ; 0-9
                    %x41-5A / ; A-Z
                    %x5F /    ; _
                    %x61-7A / ; a-z
                    %x7F-10FFFF
```
