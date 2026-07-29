# Variant Fields

Coggy does not have file-level enumerations. Instead, Algebraic Data Types are modeled by declaring Variant Fields inside a `Data` type.

A variant field encapsulates a hidden integer tag, an optional base value, and an optional variant-specific memory payload.

```EBNF
VariantFieldDecl ::= [FieldModifier] Identifier [":" Type ["=" Expression]] "{" Variant+ "}"
Variant          ::= PublicId ["(" PayloadList ")"] ["=" Expression] ","?
PayloadList      ::= PayloadField ("," PayloadField)*
PayloadField     ::= LocalId ":" Type ["=" Expression]
```

* **Optional Base Type**:
  * If the `: Type` is **omitted**, the field acts purely as a state flag. The compiler implicitly assigns it an `Int` base type to represent the variant tag.
  * If the `: Type` is **provided**, the compiler allocates an `Int` tag PLUS memory for the specified base type, PLUS memory for the largest payload.
* **Variant Resolution**:
  * If the field has a base type, every variant MUST resolve to a value of that type.
  * If a variant explicitly declares `= Expression`, that value is used.
  * If a variant omits `= Expression`, the field's global `Expression` is used.
* **Payload Defaults**: Payload fields inside a variant MAY specify default values (`= Expression`). These are evaluated and assigned if the variant is instantiated without explicitly providing those fields.
* **Shared Members and Methods**: Because the variant is just a field, any shared state (like `X`, `Y`) or methods belong to the enclosing `Data` type.
* **Name Collision**: A variant field shares the member namespace of the enclosing `Data` type. A variant field MUST NOT have the same name as any other field, property, or method within the same type.

```C#
Option<T> 
{
    State 
    {
        None,
        Some(value: T)
    }

    Unwrap(@) throws -> T 
    {
        if @.State is .Some(val) => return val;
        throw Exception.New("Unwrapped None");
    }
}

SomeData
{
    X: Int;
    Y: Int;

    // Variant field with base type 'String'
    VariedField: String 
    {
        Wow = "Wow!",
        Hmm(x: Int, y: Float = 0.0f) = "Hmm...", // Payload 'y' has a default value
        BadVariant(code: Int) = "Error"
    }

    // Variant field with a default value
    Status: String = "Mmm" 
    {
        One,             // Inherits "Mmm"
        Two = "Cool"     // Overrides with "Cool"
    }
}
```

## Instantiation

Variant fields are initialized using the explicit `::` scope resolution operator based on the field's name, or via the leading dot (`.`) syntax if the type can be contextually inferred.

```C#
some := SomeData 
{ 
    X: 0, 
    Y: 0, 
    // 'y' is omitted, relying on its default value 0.0f
    // Using contextual implicit resolution (leading dot)
    VariedField: .Hmm { x: 10 },
    
    // Using explicit full path
    Status: SomeData::Status::One {}
};

noneVal := Option<Int> { State: .None {} };
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
if some.VariedField is .Hmm(x, y) 
{
    x = 5;
    y = 5.0f;
    Console.WriteLine(some.VariedField); // Prints "Hmm..."
}
```

## Casting Operators

To force refinement outside of an `is` block, explicit casting operators MUST be used.

* **`as` (Throwing Cast)**: Asserts the variant. Throws `CastException` if the active tag does not match.
* **`as!` (Unchecked Cast)**: Forces the compiler to treat the field as the specified variant WITHOUT runtime tag validation.
* **`as?` (Option Cast)**: Evaluates the tag and returns an Option to the refined payload.

```C#
// Throwing cast
hmm := some.VariedField as .Hmm;
Console.WriteLine(hmm.x);

// Unchecked cast
fastHmm := some.VariedField as! .Hmm;

// Pattern Matching
match some.VariedField 
{
    .Wow => Console.WriteLine("No payload"),
    .Hmm(x, y) => Console.WriteLine(x),
    _ => Console.WriteLine("Fallback")
}
```
