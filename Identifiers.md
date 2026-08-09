# Identifiers

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

* MUST consist exclusively of ASCII letters (`A-Z`, `a-z`), digits (`0-9`), and underscores (`_`).
* MUST NOT contain Unicode characters.
* MUST enforce identifier usage based strictly on lexical formatting and declaration context.

## Public Identifiers

* MUST start with an uppercase letter and contain only letters or digits.
* MUST NOT end with an underscore.
* MUST designate globally accessible `Data` types, properties, methods, variants, and fields.

## Internal Identifiers

* MUST start with an uppercase letter, contain only letters or digits, and terminate with exactly one underscore.
* MUST designate members accessible exclusively within the declaring compiled package.

## Private Identifiers

* MUST start with a single underscore, followed by a lowercase letter, and contain only letters or digits.
* MUST designate type-scoped private members or file-scoped private declarations.

## Local Identifiers

* MUST start with a lowercase letter and contain only letters or digits.
* MUST designate local variables, method parameters, and pattern-matching bindings.
* MUST NOT use `PublicId`, `InternalId`, or `PrivateId` formats for local scope bindings.

```C#
DataName
Engine_
_cacheSize
index

// Invalid:
1stValue
_Data
Data__
user_age
```

## Keywords Restriction

* MUST NOT exactly match any reserved keyword.
* MAY escape a reserved keyword to function as an identifier by enclosing it in backticks (`` ` ``).
* MUST strip backticks during lexing; backticks MUST NOT form part of the compiled symbol name.

```C#
`match` := 10;
Console.WriteLine(`match`);

ExtAPI.`spawn`();
```
