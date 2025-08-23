---
title: Raw String Literals
jep: 12a
author: Michael Downling, Maxime Labelle, Richard Gibson
status: accepted
created: 2022-11-19
---

# Raw String Literals

## Abstract

This JEP proposes the following modifications to JMESPath to improve the
usability of the language and ease the implementation of parsers:

- Addition of a **raw string literal** to JMESPath. This will allow for the
  direct expression of string content that would otherwise be interpreted as
  JSON escape sequences (e.g., `'\n'`, `'\r'`, `'\u005C'`).

- Deprecation of the current literal parsing behavior that allows unquoted text
  to be parsed as a JSON string. This removes an ambiguity in the JMESPath
  grammar and helps ensure consistency among implementations.

This proposal seeks to add the following syntax to JMESPath:

```jmespath
'foobar'
'foo\'bar'
`bar` -> Parse error
```

---

## Motivation

Raw string literals are provided in
[various programming languages](https://en.wikipedia.org/wiki/String_literal#Raw_strings)
to prevent language-specific interpretation (i.e., JSON parsing) and remove the
need for excessive escaping. This avoids a common problem called
[leaning toothpick syndrome (LTS)](https://en.wikipedia.org/wiki/Leaning_toothpick_syndrome),
where strings become unreadable due to the proliferation of escape characters
needed to avoid delimiter collision (e.g., `"\\\\\\"`).

When evaluating a JMESPath expression, it is often necessary to use string
literals that are not extracted from the data being evaluated but are instead a
static part of the expression. String literals are useful in many contexts, most
notably when invoking functions or constructing multi-select lists and hashes.

The current JMESPath specification uses backticks (`` ` ``) for literals, which
are parsed as JSON. For example, the following expression produces the
three-character string "foo":

```jmespath
`"foo"`
```

### Current Literal Behavior

The current specification is functionally equivalent to the example above but
also allows the quotes to be elided from the JSON literal:

```jmespath
`foo`
```

These literals are parsed according to the JSON standard
([RFC 4627](https://www.ietf.org/rfc/rfc4627.txt)), which expands Unicode escape
sequences, newlines, and several other escape sequences. For example, the
Unicode escape `\u002B` is expanded into `+`:

```jmespath
`"foo\u002B"` -> "foo+"
```

To prevent an escape sequence from being expanded, you must escape the backslash
itself:

```jmespath
`"foo\\u002B"` -> "foo\u002B"
`foo\\u002B` -> "foo\u002B"
```

While this allows for providing literal strings, it presents several problems:

1.  **Performance:** It incurs an unnecessary JSON parsing penalty.
2.  **Complexity:** It requires the cognitive overhead of escaping characters to
    represent data literally, which can lead to LTS. If the target string itself
    contains escapes, the number of backslashes doubles, quickly becoming
    unmanageable.
3.  **Ambiguity:** It introduces an ambiguous rule to the JMESPath grammar that
    requires prose-based clarification, leading to potential inconsistencies
    between parser implementations.

The relevant grammar rules are currently defined as follows:

```abnf
literal = "`" json-text "`"
literal =/ "`" 1*(unescaped-literal / escaped-literal) "`"
unescaped-literal = %x20-21 /     ; space !
                      %x23-5B /     ; # - [
                      %x5D-5F /     ; ] ^ _
                      %x61-7A /     ; a-z
                      %x7C-10FFFF   ; |}~ ...
escaped-literal     = escaped-char / (escape %x60)

; The following is a simplified subset of the JSON ABNF
json-text = ws json-value ws
json-value = false / null / true / json-object / json-array /
             json-number / json-quoted-string
false = %x66.61.6c.73.65    ; false
null  = %x6e.75.6c.6c      ; null
true  = %x74.72.75.65      ; true
json-quoted-string = %x22 1*(unescaped-char / escaped-char) %x22
; ...etc.
```

The `literal` rule is ambiguous because the characters matched by
`unescaped-literal` are a superset of those that can start a `json-text` value.
This allows a given input to be matched by either rule, creating ambiguity.

---

## Rationale

The allowance of elided quotes around JSON string literals requires special-case
handling in JMESPath parsers. This aspect of the specification cannot be
described unambiguously in a context-free grammar and is a common source of
implementation errors.

Parsing these literals is overly complicated. The steps required to parse a JSON
literal in JMESPath are:

1.  When a `` ` `` token is encountered, begin parsing a literal.
2.  Collect all characters between the opening and closing `` ` `` tokens into a
    variable (e.g., `$lexeme`), respecting escaped backticks (`\` \`\`).
3.  Create a temporary value, `$temp`, by trimming leading and trailing
    whitespace from `$lexeme`. (This behavior is currently undocumented but is
    required by the
    [JMESPath compliance tests](https://github.com/jmespath-community/jmespath.test/blob/main/tests/syntax.json).
4.  If `$temp` can be parsed as valid JSON, use the parsed result as the value
    for the literal token.
5.  If `$temp` cannot be parsed as valid JSON, treat the original `$lexeme` as a
    string. This is equivalent to wrapping `$lexeme` in double quotes and
    parsing the result as a JSON string. Therefore, `` `foo` `` is equivalent to
    `` `"foo"` ``, and `` `[1, ]` `` is equivalent to `` `"[1, ]"` ``.

It is reasonable to assume that the most common use case for a literal in a
JMESPath expression is to provide a simple string value, either as a function
argument or as a value in a multi-select list or hash. The original intent of
eliding quotes was to make this common case easier.

This proposal posits that this convenience introduces excessive complexity and
ambiguity. A dedicated raw string literal is a clearer, more robust solution.

---

## Specification

A **raw string literal** is a value that begins and ends with a single quote
(`'`). It preserves all characters literally, including backslashes, except for
`\'` and `\\`, which are unescaped to a single quote and a backslash,
respectively.

### Examples

Here are several examples of valid raw string literals and their parsed values:

- A basic raw string, representing the seven-character string "foo bar":

  ```jmespath
  'foo bar'
  ```

- A raw string with an escaped single quote, representing the seven-character
  string "foo'bar":

  ```jmespath
  'foo\'bar'
  ```

- A raw string with an escaped backslash, representing the seven-character
  string "foo\\bar":

  ```jmespath
  'foo\\bar'
  ```

- A raw string that contains newlines:

  ```jmespath
  'foo
  bar
  baz!'
  ```

  The expression above represents the multi-line string:

  ```
  foo
  bar
  baz!
  ```

- A raw string that contains a preserved escape sequence, representing the
  eight-character string "foo\\nbar":

  ```jmespath
  'foo\nbar'
  ```

### ABNF

The following ABNF grammar rules will be added:

```abnf
expression =/ raw-string
raw-string = "'" *raw-string-char "'"
raw-string-char   = unescaped-raw-string-char /
                    preserved-escape /
                    raw-string-escape
unescaped-raw-string-char = %x00-26 /     ; ␀ through '&' (precedes ')
                            %x28-5B /     ; '(' through '[' (precedes \)
                            %x5D-10FFFF   ; ']' and all following code points
preserved-escape  = escape (%x00-26 / %x28-5B / %x5D-10FFFF)
raw-string-escape = escape ("'" / escape) ; U+0027 ' or U+005C \
```

These rules allow any character inside a raw string. Only a backslash followed
by a single quote or another backslash is interpreted as an escape sequence. All
other backslash sequences (e.g., `\n`, `\t`) are preserved literally.

In addition, the `literal` rule in the ABNF will be simplified to remove
ambiguity:

```abnf
literal = "`" json-text "`"
```

This change requires that any content within backticks must be a valid,
self-contained JSON document.

---

## Impact

This is a **breaking change**. Existing JMESPath expressions that use unquoted
literals (e.g., `` `foo` ``) will no longer be valid. Users must update these
expressions to either use valid JSON within the backticks (e.g., `` `"foo"` ``)
or, preferably, adopt the new raw string syntax (e.g., `'foo'`).

To accommodate legacy JMESPath implementations during a transition period, all
compliance tests for unquoted literals should be moved to a JEP-12-specific test
suite. This allows implementations to explicitly opt-in or out of testing for
the deprecated behavior.

---

## Alternative Approaches

### Leave As-Is

The simplest alternative is to make no changes. This would avoid a breaking
change, and users could continue to use backslash-escaping to handle special
characters. However, the goal of this proposal is not to add functionality but
to make JMESPath easier to use, reason about, and implement. The current
behavior of literal parsing is ambiguous and can lead to subtle differences
between implementations. For example:

```jmespath
`[1`
```

One implementation might parse this as the string `"[1"`, while another might
raise a parse error because the content begins like, but is not, valid JSON.
Requiring valid JSON within backticks removes this ambiguity entirely.

### Disallow Single Quotes in a Raw String

An alternative could be to forbid single quotes within a raw string literal,
removing the need for `\'`. While this would simplify the grammar, it would
severely limit the feature's utility, forcing users back to JSON literals for
any string containing a single quote.

### Use a Customizable Delimiter

Languages like Lua, D, and C++11 allow custom delimiters for raw strings, such
as `[==[foo=bar]==]`. This approach is very flexible and eliminates the need for
any escaping. However, it requires a stateful parser that cannot be expressed in
a simple context-free grammar, adding significant complexity to implementations.

The addition of the single-quoted string literal proposed in this JEP does not
preclude the future addition of a more advanced delimited string syntax.
