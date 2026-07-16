# Identifiers

Identifiers are names used to denote types, variables, fields, methods, properties, data, and packages.

* **Only ASCII**: Identifiers MUST strictly consist of ASCII letters (`A-Z`, `a-z`), digits (`0-9`), and underscores (`_`).
* **No Unicode**: Unicode characters are STRICTLY FORBIDDEN in identifiers.

```EBNF
Identifier     ::= PublicId | InternalId | PrivateId | LocalId

PublicId       ::= Uppercase Alphanumeric* (excluding those ending with "_")
InternalId     ::= Uppercase Alphanumeric* "_"
PrivateId      ::= "_" Lowercase Alphanumeric*
LocalId        ::= Lowercase Alphanumeric*

Uppercase      ::= [A-Z]
Lowercase      ::= [a-z]
Digit          ::= [0-9]
Alphanumeric   ::= Uppercase | Lowercase | Digit
```

## Identifier Categories

The compiler STRICTLY enforces where each category of identifier can be used based on its declaration context.

### Public Identifiers

* **Format**: Starts with an uppercase letter, followed by letters or digits. MUST NOT end with an underscore.
* **Usage**: Used for globally accessible Data types, Enums, properties, methods, and shared fields.

### Internal Identifiers

* **Format**: Starts with an uppercase letter, followed by letters or digits, and strictly ends with a single underscore.
* **Usage**: Used for members accessible only within the same compiled Package.

### Private Identifiers

* **Format**: Starts with a single underscore, followed strictly by a lowercase letter, and then letters or digits.
* **Usage**: Used for type-scoped private fields/methods or file-scoped private declarations.

### Local Identifiers

* **Format**: Starts with a lowercase letter, followed by letters or digits.
* **Usage**: Used STRICTLY for local variables, method parameters, and pattern-matching bindings. Local variables MUST NOT use `PublicId`, `InternalId`, or `PrivateId` formats.

```C#
// Valid:
DataName        // PublicId
Engine_         // InternalId
_cacheSize      // PrivateId
index           // LocalId

// Invalid:
1stValue        // ERROR: Cannot start with a digit
_Data           // ERROR: PrivateId must start with '_' followed by lowercase
Data__          // ERROR: Multiple underscores are forbidden
user_age        // ERROR: Underscores in the middle are forbidden (snake_case is not allowed)
```

## Keywords Restriction

An identifier MUST NOT exactly match any of Coggy's reserved keywords.

* **Escaping**: If interoperability requires using a keyword as an identifier, the identifier MUST be enclosed in backticks (`` ` ``) during declaration and usage.
* **Symbol Name**: The backticks are strictly a lexer construct, they are stripped during compilation and are NOT part of the final symbol name.

```C#
// Variable named 'match' using keyword escape
`match` := 10;
Console.WriteLine(`match`);

// Calling an external function named 'spawn'
ExtAPI.`spawn`();
```
