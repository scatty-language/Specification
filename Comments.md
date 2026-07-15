# Comments

Comments are non-executing text ignored by the compiler. They are stripped during lexical analysis and produce no tokens.

```EBNF
Comment        ::= LineComment | BlockComment
LineComment    ::= "//" [^NewLine]*
BlockComment   ::= "/*" BlockChar* "*/"
BlockChar      ::= [^"*"] | "*" [^"/"]
```

## Line Comments

A line comment begins with `//` and extends to the end of the physical line.

* **Termination**: The comment terminates at the next LineFeed (`\n`). The line terminator itself is not consumed as part of the comment.
* **Nesting**: `//` comments do not nest. Any `//` after the opening delimiter is treated as comment text.

```C#
// This is a line comment
x := 1; // Valid: trailing comment
```

## Block Comments

A block comment begins with `/*` and ends with the first subsequent `*/`.

* **Non-Nesting**: Block comments do NOT nest. The first `*/` encountered closes the comment, regardless of how many `/*` precede it.
* **Multi-Line**: A block comment MAY span multiple physical lines. Line terminators inside the comment are consumed and do not affect token positions outside the comment.
* **Unterminated Comment**: A block comment that reaches end-of-file without a closing `*/` is a compilation error.

```C#
/* Valid: single-line block comment */
x := 2;

/* Valid:
   multi-line
   block comment */
y := 3;

/* Invalid: nested comment
   /* inner */          // <- This closes the outer comment
   remaining text */    // <- Error: unexpected '*/'
```

## Interaction with String Literals

Comment delimiters inside a string or character literal are literal text and do NOT start a comment.

```C#
s := "/* not a comment */";   // Valid: entire string is literal text
t := "// also not a comment"; // Valid
```
