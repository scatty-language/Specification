# Comments

```EBNF
Comment        ::= LineComment | BlockComment
LineComment    ::= "//" (SourceChar - NewLine)*
BlockComment   ::= "/*" (BlockChar | BlockComment)* "*/"
BlockChar      ::= /* Any sequence of characters that does not contain "/*" or "*/" */
SourceChar     ::= /* Any valid UTF-8 character */
```

* MUST strip all comments during lexical analysis.
* MUST NOT generate tokens from commented text.

## Line Comments

* MUST initiate a line comment upon encountering the `//` sequence outside string, character, or rune literals.
* MUST terminate a line comment at the next `NewLine` sequence without consuming the line terminator.
* MUST NOT parse nested line comments. Subsequent `//` sequences on the same line MUST be treated as comment text.

## Block Comments

* MUST initiate a block comment upon encountering the `/*` sequence outside string, character, or rune literals.
* MUST terminate a block comment upon encountering the matching `*/` sequence.
* MUST support nested block comments by requiring every nested `/*` delimiter to be balanced by a corresponding `*/` delimiter.
* MAY span multiple lines.
* MUST NOT permit an unclosed block comment at the end of a file.

## Interaction with Literals

* MUST treat comment delimiters within string, character, or rune literals as literal characters.
