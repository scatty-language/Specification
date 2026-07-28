# Comments

Comments are non-executing text ignored by the compiler. They are stripped during lexical analysis and produce no tokens.

```EBNF
Comment        ::= LineComment | BlockComment
LineComment    ::= "//" (SourceChar - NewLine)*
BlockComment   ::= "/*" (BlockChar | BlockComment)* "*/"
BlockChar      ::= /* Any sequence of characters that does not contain "/*" or "*/" */
SourceChar     ::= /* Any valid UTF-8 character */
```

## Line Comments

A line comment begins with `//` and extends to the end of the physical line.

* **Termination**: The comment terminates at the next `NewLine`. The line terminator itself is NOT consumed as part of the comment.
* **Nesting**: `//` comments do not nest. Any `//` sequence encountered after the opening delimiter is treated as standard comment text.

```C#
// Line comment
x := 1;
```

## Block Comments

A block comment begins with `/*` and ends with the matched `*/` sequence.

* **Nesting**: Block comments natively nest. Every opening `/*` sequence MUST be balanced by a corresponding closing `*/` sequence.
* **Multi-Line**: A block comment MAY span multiple physical lines. Line terminators inside the comment are consumed and DO NOT affect token positions outside the comment.
* **Unterminated Comment**: A block comment that reaches end-of-file without all opening `/*` delimiters being properly closed by `*/` is a FORBIDDEN.

```C#
/* Single-line block comment */
x := 2;

/* Multi-line
   block comment */
y := 3;

/* Valid nested comment
   /* inner comment */          
   remaining text 
*/    
```

## Interaction with String Literals

Comment delimiters appearing inside a valid `String`/`Char`/`Rune` literal are treated as literal text and DO NOT initiate a comment.

```C#
s := "/* not a comment */";
t := "// also not a comment";
```
