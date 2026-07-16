# Variant Fields

Coggy does not have module-level enumerations. Instead, Algebraic Data Types are strictly modeled by declaring Variant Fields inside a `Data` type.

A variant field encapsulates a hidden integer tag, an optional base value, and an optional variant-specific memory payload.

```EBNF
VariantFieldDecl ::= "%" Identifier [":" Type] ["=" Expression] "{" Variant+ "}"
Variant          ::= PublicId ["(" PayloadList ")"] ["=" Expression] ","?
PayloadList      ::= PayloadField ("," PayloadField)*
PayloadField     ::= LocalId ":" Type ["=" Expression]
```

* **Optional Base Type**:
  * If the `: Type` is **omitted**, the field acts purely as a state flag. The compiler implicitly assigns it an `int` base type to represent the variant tag.
  * If the `: Type` is **provided**, the compiler allocates an `int` tag PLUS memory for the specified base type, PLUS memory for the largest payload.
* **Variant Resolution**:
  * If the field has a base type, every variant MUST resolve to a value of that type.
  * If a variant explicitly declares `= Expression`, that value is used.
  * If a variant omits `= Expression`, the field's global `Expression` is used.
* **Payload Defaults**: Payload fields inside a variant MAY specify default values (`= Expression`). These are evaluated and assigned if the variant is instantiated without explicitly providing those fields.
* **Shared Members and Methods**: Because the variant is just a field, any shared state (like `X`, `Y`) or methods belong to the enclosing `Data` type.
* **Name Collision**: A variant field shares the same namespace as nested data types. A variant field MUST NOT have the same name as a nested `Data` declaration within the same scope, and vice versa. Shadowing is strictly a compilation error.

```C#
Option<T> 
{
    %State 
    {
        None,
        Some(value: T)
    }

    Unwrap(@) throws -> T 
    {
        if @.State is Option::State::Some => return @.State.value;
        throw #Exception.New("Unwrapped None");
    }
}

SomeData
{
    X: int;
    Y: int;

    // Variant field with base type 'string'
    %VariedField: string 
    {
        Wow = "Wow!",
        Hmm(x: int, y: float = 0.0f) = "Hmm...", // Payload 'y' has a default value
        BadVariant(code: int) = "Error"
    }

    // Variant field with a default value
    %Status: string = "Mmm" 
    {
        One,             // Inherits "Mmm"
        Two = "Cool"     // Overrides with "Cool"
    }

    Status { } // ERROR: Cannot declare nested Data with the same name as a variant field
}
```

## Instantiation

Variant fields are initialized using the `::` scope resolution operator based on the field's name. The `%` modifier is NOT used during access or instantiation.

```C#
some := SomeData 
{ 
    X: 0, 
    Y: 0, 
    // 'y' is omitted, relying on its default value 0.0f
    VariedField: SomeData::VariedField::Hmm { x: 10 },
    Status: SomeData::Status::One {}
};

noneVal := Option<int> { State: Option::State::None {} };
```

## Access

Because variant fields implicitly represent their base type, direct access evaluates to the base value. Accessing a variant-specific payload requires type refinement.

### Base Type Access

Direct access (`instance.Field`) returns the base value without evaluating the payload.

```C#
Console.WriteLine(some.VariedField); // Prints: "Hmm..."
```

### `is` Operator

Checks the active variant tag. If used in a condition, it implicitly refines the field's type within the block, granting access to payload variables.

```C#
if some.VariedField is SomeData::VariedField::Hmm 
{
    // Inside this scope, 'VariedField' is refined
    some.VariedField.x = 5;
    some.VariedField.y = 5.0f;
    Console.WriteLine(some.VariedField); // Prints "Hmm..."
}
```

## Casting Operators

To force refinement outside of an `is` block, explicit casting operators MUST be used.

* **`as` (Throwing Cast)**: Asserts the variant. Throws `#CastException` if the active tag does not match.
* **`as!` (Unchecked Cast)**: Forces the compiler to treat the field as the specified variant WITHOUT runtime tag validation.
* **`as?` (Option Cast)**: Evaluates the tag and returns an Option to the refined payload.

```C#
// Throwing cast
hmm := some.VariedField as SomeData::VariedField::Hmm;
Console.WriteLine(hmm.x);

// Unchecked cast
fastHmm := some.VariedField as! SomeData::VariedField::Hmm;

// Pattern Matching
match some.VariedField 
{
    .Wow => Console.WriteLine("No payload"),
    .Hmm => Console.WriteLine(some.VariedField.x),
    _ => Console.WriteLine("Fallback")
}
```
