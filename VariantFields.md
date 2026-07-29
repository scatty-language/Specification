# Variant Fields

Coggy does not have file-level enumerations. Instead, Algebraic Data Types are modeled by declaring Variant Fields inside a `Data` type.

A variant field encapsulates a hidden discriminant tag, an optional base value, and an optional variant-specific memory payload.

```EBNF
VariantFieldDecl ::= [FieldModifier] Identifier [":" Type ["=" Expression]] ["::" TagType] "{" Variant+ "}"
Variant          ::= PublicId ["(" PayloadList ")"] ["=" Expression] ["::" Expression] ","?
PayloadList      ::= PayloadField ("," PayloadField)*
PayloadField     ::= LocalId ":" Type ["=" Expression]
TagType          ::= Identifier
```

* **Optional Base Type**:
  * If the `: Type` is omitted, the field acts purely as a state flag.
  * If the `: Type` is provided, the compiler allocates a discriminant tag PLUS memory for the specified base type, PLUS memory for the largest payload.
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

## Explicit Tags

By default, the compiler intrinsically manages the hidden variant tag as an `Int`, automatically assigning sequential values starting from `0`.

Developers MAY explicitly define the tag's underlying type and its exact values.

* **Tag Type**: Declared after the optional base type and default value using `:: TagType`. It MUST resolve to a scalar integer primitive (e.g., `Byte`, `Int`, `Usize`).
* **Tag Values**: Individual variants MAY declare their explicit tag value using `:: Expression`. The expression MUST be a compile-time constant matching the Tag Type.
* **Auto-Increment**: If a variant omits the explicit tag value, the compiler automatically assigns it the value of the previous variant plus `1` (or `0` for the first variant).

```C#
Connection 
{
    // Base type is 'String', default is "Idle"
    // Discriminant tag is explicitly a 'Byte'
    State: String = "Idle" :: Byte 
    {
        Disconnected = "Offline" :: 0x00,
        
        // Base value inherited ("Idle"), Tag explicitly set to '0x01'
        Connecting :: 0x01,
        
        // Tag auto-increments to '0x02'
        Connected = "Online", 
        
        // Tag explicitly set to '0xFF', with payload
        Error(code: Int) = "Fault" :: 0xFF 
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
