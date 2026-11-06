# Identifiers

* MUST consist exclusively of ASCII letters (`A-Z`, `a-z`), digits (`0-9`), and underscores (`_`).
* MUST NOT contain Unicode characters.
* MUST enforce identifier usage based on lexical formatting and declaration context.

## Public Identifiers

* MUST start with an uppercase letter and contain only letters or digits.
* MUST NOT end with an underscore.
* MUST designate globally accessible `Data` types, properties, methods, variants, and fields.

## Internal Identifiers

* MUST start with an uppercase letter, contain only letters or digits, and terminate with exactly one underscore.
* MUST designate members accessible exclusively within the declaring compiled package.

## Private Identifiers

* MUST start with a single underscore, followed by a lowercase letter, and contain only letters or digits.
* MUST designate type-scoped private members or file-scoped private declarations accessible within the declaring file and its narrower conditional compilation files sharing the same `<BaseName>`.

## Local Identifiers

* MUST start with a lowercase letter and contain only letters or digits.
* MUST designate local variables, method parameters, and pattern-matching bindings.
* MUST NOT use `PublicId`, `InternalId`, or `PrivateId` formats for local scope bindings.

```scatty
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

```scatty
`match` := 10;
Console.WriteLine(`match`);

ExtAPI.`spawn`();
```
