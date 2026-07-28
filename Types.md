# Types

Coggy employs a deterministically parsed type system. A complete type signature is formed by a core base type wrapped in optional prefix and postfix modifiers.

```EBNF
Type            ::= PrefixModifier* BaseType PostfixModifier*
BaseType        ::= Identifier ["<" TypeArgs ">"] | TupleType | "(" Type ")"
PrefixModifier  ::= "[]" | "[~]" | "[" Decimal "]"
PostfixModifier ::= "[]" | "[~]" | "[" Decimal "]" | "[$]" | "[$" Type "]" | "?" | "*"
```

## Type Modifier Precedence

1. **Postfix Precedence**: Postfix modifiers ALWAYS bind with strictly higher precedence than prefix modifiers.
2. **Right-to-Left / Left-to-Right Binding**:
   * Multiple postfix modifiers bind left-to-right (from the base type outwards).
   * Multiple prefix modifiers bind right-to-left (from the base type outwards).
3. **Explicit Override**: Parentheses `()` MAY be used around any valid `Type` to explicitly override the default binding precedence.

```C#
// Postfix evaluated first
// Parsed as: [] (Int?)
// Meaning: An SoA array of 'Option<Int>'
a: []Int? = [];

// Explicit grouping
// Parsed as: ([]Int)?
// Meaning: An 'Option' containing an SoA array of 'Int'
b: ([]Int)? = .None {};

// Multiple postfixes
// Parsed as: (Int[])?
// Meaning: An 'Option' containing a standard Heap Array of 'Int'
c: Int[]? = .None {};

// Combined collection modifiers
// Parsed as: [~] (Int[])
// Meaning: A dynamic SoA List of standard Heap Arrays
d: [~]Int[] = [~]Int[].New();
```
