---
description: "Raw string literals can contain any arbitrary text without the need for special escape sequences. You begin and end a raw string literal with a minimum of three double-quote characters."
title: "Raw string literals - \"\"\""
f1_keywords:
  - "RawStringLiteral_CSharpKeyword"
ms.date: 12/08/2022
---
# Raw string literal text - `"""` in string literals

A raw string literal starts and ends with a minimum of three double quote (`"`) characters:

:::code language="csharp" source="./snippets/raw-string-literal.cs" id="SingleLine":::

Raw string literals can span multiple lines:

:::code language="csharp" source="./snippets/raw-string-literal.cs" id="MultiLine":::

The following rules govern the interpretation of a multi-line raw string literal:

- The opening quotes must be the last non-comment token on its respective line, and the closing quote must be the first non-comment token on its respective line.
- Any whitespace to the left of the closing quotes is removed from all lines of the raw string literal.
- Whitespace following the opening quote on the same line is ignored.
- Whitespace only lines following the opening quote are included in the string literal.
- If a whitespace precedes the end delimiter on the same line, the exact number and kind of whitespace characters (for example, spaces vs. tabs) must exist at the beginning of each content line. Specifically, a space does not match a horizontal tab, and vice versa.
- The newline before the closing quotes isn't included in the literal string.

You may need to create a raw string literal that has three or more consecutive double-quote characters. Raw string literals can start and end with a sequence of at least three double-quote characters. When your string literal contains three consecutive double-quotes, you start and end the raw string literal with four double quote characters:

:::code language="csharp" source="./snippets/raw-string-literal.cs" id="MoarQuotes":::

If you need to start or end a raw string literal with quote characters, use the multi-line format:

:::code language="csharp" source="./snippets/raw-string-literal.cs" id="InitialQuotes":::

Raw string literals can also be combined with [interpolated strings](./interpolated.md#interpolated-raw-string-literals) to embed the `{` and `}` characters in the output string. You use multiple `$` characters in an interpolated raw string literal to embed `{` and `}` characters in the output string without escaping them.

The raw string literal's content must not contain a set of contiguous `"` characters whose length is equal to or greater than the raw string literal delimiter length. For example, the strings `"""" """ """"` and `""""""" """""" """"" """" """ """""""` are well-formed. However, the strings `""" """ """` and `""" """" """` are ill-formed

Raw string literals were introduced in C# 11.

## See also

- [C# special characters](./index.md)
- [C# string interpolation](./interpolated.md)
- [Raw string literals feature specification](~/_csharplang/proposals/csharp-11.0/raw-string-literal.md)
