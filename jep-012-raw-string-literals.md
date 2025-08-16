---
title: Raw String Literals
jep: 12
created: 2015-04-09
author: Michael Dowling
status: obsoleted
obsoleted_by: 12a
---

# Raw String Literals

## Abstract

This JEP proposes modifications to JMESPath to improve usability and parser
implementation consistency by:

- Adding **raw string literals** that contain unprocessed strings without JSON
  escape sequence interpretation (e.g., "\\n", "\\r", "\\u005C")
- Deprecating the current literal parsing behavior that allows unquoted JSON
  strings, eliminating grammar ambiguity and ensuring implementation consistency

The proposal introduces the following syntax:

```
'foobar'
'foo\'bar'
`bar` → Parse error/warning (implementation specific)
```

## Motivation

Raw string literals exist in
[various programming languages](https://en.wikipedia.org/wiki/String_literal#Raw_strings)
to prevent language-specific interpretation and avoid
[leaning toothpick syndrome (LTS)](https://en.wikipedia.org/wiki/Leaning_toothpick_syndrome)—where
strings become unreadable due to excessive escape characters.

When evaluating JMESPath expressions, string literals are often needed as static
components rather than values extracted from input data. These literals are
particularly useful when invoking functions or constructing multi-select lists
and hashes.

Consider this expression that returns the character count of "foo":

```
`"foo"`
```

This is functionally equivalent to:

```
`foo`
```

These string literals are parsed using JSON according to
[RFC 4627](https://www.ietf.org/rfc/rfc4627.txt), which expands Unicode escape
sequences, newline characters, and other escape sequences.

For example, the escaped Unicode value `\u002B` is expanded to `+`:

```
`"foo\u002B"` → "foo+"
```

To prevent expansion, escape sequences must themselves be escaped:

```
`"foo\\u002B"` → "foo\u002B"
`foo\\u002B` → "foo\u002B"
```

This approach creates three significant problems:

1. Incurs unnecessary JSON parsing overhead

2. Requires cognitive effort to escape escape characters (leading to LTS),
   especially problematic when the string is meant for another language that
   also uses `\` as an escape character

3. Introduces ambiguous grammar rules requiring prose-based specification to
   resolve parser implementation differences

The current literal grammar rules are:

```
literal = "`" json-value "`"
literal =/ "`" 1*(unescaped-literal / escaped-literal) "`"
unescaped-literal = %x20-21 /       ; space !
                        %x23-5B /   ; # - [
                        %x5D-5F /   ; ] ^ _
                        %x61-7A     ; a-z
                        %x7C-10FFFF ; |}~ ...
escaped-literal   = escaped-char / (escape %x60)
```

The `literal` rule is ambiguous because `unescaped-literal` includes all
characters that `json-value` matches, allowing any valid JSON value to match
either rule.

### Rationale

Implementing JMESPath parsers requires special-case handling for JSON literals
due to the allowance of unquoted string literals (e.g., ` ```foo``` `). This
aspect cannot be unambiguously described in a context-free grammar and is a
common source of parser implementation errors.

Parsing JSON literals currently requires these steps:

1. On encountering a `` ` `` token, begin parsing a JSON literal

2. Collect characters between opening and closing `` ` `` tokens (including
   escaped `` ` `` as ``\` ``) into a variable (e.g., `$lexeme`)

3. Create `$temp` by removing leading/trailing whitespace from `$lexeme`
   (undocumented but required by
   [JMESPath compliance tests](https://github.com/jmespath/jmespath.test/blob/c532a20e3bca635fb6ca248e5e955e1bd146a965/tests/syntax.json#L592-L606))

4. If `$temp` parses as valid JSON, use the parsed result

5. If `$temp` fails JSON parsing, wrap `$lexeme` in double quotes and parse as
   JSON string (making ` ```foo``` ` equivalent to ` ```"foo"``` `)

The most common use case for JSON literals in JMESPath is providing string
values to function arguments or multi-select structures. Allowing quote elision
was intended to simplify string provision, but this proposal argues that adding
proper string literals would be more effective.

## Specification

A raw string literal begins and ends with a single quote, doesn't interpret
escape characters, and allows escaped single quotes to avoid delimiter
collision.

### Examples

- Basic raw string literal, parsed as `foo bar`:

```
'foo bar'
```

- Escaped single quote, parsed as `foo'bar`:

```
'foo\'bar'
```

- Raw string literal containing newlines:

```
'foo
bar
baz!'
```

This would be parsed exactly as written, preserving newlines:

```
foo
bar
baz!
```

- Raw string literal containing escape characters, parsed as `foo\nbar`:

```
'foo\nbar'
```

### ABNF

The following grammar rules will be added, valid wherever expressions are
allowed:

```
raw-string        = "'" *raw-string-char "'"
; Matches any character except "\"
raw-string-char   = (%x20-26 / %x28-5B / %x5D-10FFFF) / raw-string-escape
raw-string-escape = escape ["'"]
```

This allows any character inside raw strings, including escaped single quotes.

Additionally, the `literal` rule will be simplified to:

```
literal = "`" json-value "`"
```

## Impact

Existing JMESPath users should convert elided-quote JSON literals to use the new
string-literal syntax. Whether this conversion is mandatory depends on the
specific implementation.

Implementations may choose to support the old syntax for backward compatibility,
but should issue warnings about deprecation and potential cross-implementation
incompatibility.

To accommodate implementation variance:

- JSON literal compliance test cases involving elided quotes must be removed
- Test cases for invalid unquoted JSON values must be placed in a JEP-specific
  test suite
- Implementations supporting elided quotes can filter out JEP-specific test
  cases

## Alternative Approaches

### Leave as-is

Maintaining the current behavior avoids breaking changes but preserves the
problems this JEP aims to solve:

- The JSON parsing behavior remains ambiguous
- Parser implementations require special casing
- Minor implementation differences persist due to ambiguity

Consider this example:

```
`[1`
```

One implementation might interpret this as a JSON string `"[1"`, while another
might raise a parse error. Requiring valid JSON in literal tokens would
eliminate this ambiguity.

### Disallow Single Quotes in Raw Strings

Instead of requiring escaped single quotes (`'foo\'bar'`), an alternative could
forbid single quotes entirely in raw strings. While this would simplify the
grammar, it would severely limit raw string usability, forcing users back to
regular literals for common cases.

### Customizable Delimiters

Languages like Lua use customizable delimiters (e.g., `[==[foo=bar]==]`). While
flexible and eliminating character escaping needs, this approach cannot be
expressed in a regular grammar—parsers would need to track delimiter counts. The
proposed string literal doesn't preclude adding such features later.
