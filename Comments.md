# Comments

Comments are non-executing text ignored by the compiler. They are stripped during lexical analysis and produce no tokens.

```EBNF
Comment        ::= LineComment | BlockComment
LineComment    ::= "//" (SourceChar - NewLine)*
BlockComment   ::= "/*" BlockChar* "*/"
BlockChar      ::= (SourceChar - "*") | ("*" (SourceChar - "/"))
SourceChar     ::= /* Any valid UTF-8 character */
```

## Line Comments

A line comment begins with `//` and extends to the end of the physical line.

* **Termination**: The comment strictly terminates at the next `NewLine`. The line terminator itself is NOT consumed as part of the comment.
* **Nesting**: `//` comments do not nest. Any `//` sequence encountered after the opening delimiter is treated as standard comment text.

```C#
// Line comment
x := 1;
```

## Block Comments

A block comment begins with `/*` and ends with the first subsequent sequence `*/`.

* **Non-Nesting**: Block comments do NOT nest. The first `*/` encountered closes the comment, regardless of how many `/*` sequences precede it.
* **Multi-Line**: A block comment MAY span multiple physical lines. Line terminators inside the comment are consumed and do not affect token positions outside the comment.
* **Unterminated Comment**: A block comment that reaches end-of-file without a closing `*/` is a strict compilation error.

```C#
/* Single-line block comment */
x := 2;

/* Multi-line
   block comment */
y := 3;

/* Invalid: nested comment
   /* inner */          // Closes the outer comment
   remaining text */    // ERROR: unexpected '*/'
```

## Interaction with String Literals

Comment delimiters appearing inside a valid string or character literal are treated as literal text and do NOT initiate a comment.

```C#
s := "/* not a comment */";
t := "// also not a comment";
```
