# Inline Fields

Coggy supports transparent composition and state delegation via Inline Fields.

For Inline Fields, the compiler automatically exposes the public and internal members (or states) of the field's type directly on the enclosing `Data` type.

```EBNF
InlineFieldDecl ::= "@" FieldDecl | VariantFieldDecl
```

* **Transparent Forwarding**: Accessing a delegated member on the parent instance implicitly forwards the call to the inline field. If the forwarded method requires a `ref` receiver, the compiler automatically passes the inline field by `ref`.
* **Direct Access**: The inline field itself remains accessible via its declared identifier (e.g., `parent.Field`). Visibility rules apply to the field identifier normally.

## Inline Variant Fields

A Variant Field MAY be inlined. This allows the parent `Data` type to transparently act as a state machine without requiring nested property access for pattern matching.

* **Strict Singularity**: A `Data` declaration MUST NOT contain more than one inline variant field. Inline multiple variant fields within the same structure is FORBIDDEN.
* **Transparent State Forwarding**: Applying the `is` operator, `as` casting operators, or a `match` expression directly to the parent instance transparently evaluates the underlying active tag of the inline variant field.
* **Base Type Delegation**: If the inline variant field specifies a base `: Type`, all methods and properties of that base type are transparently forwarded to the parent instance.
* **Explicit Initialization**: During structural instantiation, the variant field MUST be assigned using its explicit identifier.

```C#
Response 
{
    @Message: String = "None" 
    {
        Success = "Wow! All fine!",
        Error(code: Int)
    }
    
    Timestamp: Int;
}

res := Response { Message: .Success, Timestamp: 12345 };

if res is .Error(code) 
{
    Console.WriteLine(code);
}

// Delegation to the base type ('String') methods
if res.Contains("Wow") 
{
    Console.WriteLine("String method forwarded!");
}
```

## Shadowing and Collisions

To ensure predictable Mutable Value Semantics and prevent accidental state corruption, strict rules govern name overlaps between the parent Data and its inline fields.

1. **Method Shadowing**: If the parent type explicitly declares a method or property with the exact same name as one exposed by an inline field, the parent's member MUST shadows (overrides) the inline field's member.
2. **Name Collision**: If the parent type explicitly declares a structural data field with the exact same name as a structural data field inside the inline type, it is a compilation error.
3. **Ambiguity Inline Fields**: If a Data type declares multiple inline fields, and two or more inline fields expose a member with the same name, accessing that member implicitly is a compilation error.
4. **Ambiguity Resolution**: To resolve ambiguity, the developer MUST explicitly declare a shadowing method or property on the parent type that manually delegates to the desired inline field.

```C#
Wrapper<T> 
{
    @List: List<T>;
    @ListB: List<T>;

    WowCount: Int;

    // Method Shadowing: overrides 'List' implicit 'Pop' method
    Pop(@) 
    {
        Console.WriteLine("Intercepted Pop");
    }

    // ERROR: 'Count' field already exists inside 'List'
    Count: Int;

    // Ambiguity Resolution
    // Accessing 'wrapper.Capacity' directly is ambiguous
    Capacity: Int => @.ListB.Capacity;
}

wrapper := Wrapper<Int> { List: List.New(), WowCount: 0, ListB: List.New() };

// Implicitly calls 'wrapper.List.Add'
wrapper.Add(0);

// Direct access remains valid
wrapper.List.Add(1);

// Shadowed method call
wrapper.Pop(); 
```

## Operator Forwarding and Primitive Types

Because primitive types are implemented as Data types within the `System` library, their operators are structurally evaluated as methods. Consequently, inline fields of primitive types transparently forward their mathematical operators to the parent type.

* **Operator Delegation**: Performing an operation like `wrapper += 10` is valid if the inline field provides the `+=` operator and the parent type does not.
* **No Implicit Subtyping**: Inline fields provide delegation, not subtyping. A Data type with an `@_field: Int` CANNOT be implicitly passed to a function expecting an `Int`.

```C#
Score 
{
    // Inline primitive field forwards math operators (+, -, etc.)
    @Value: Int;
    IsRanked: Bool;
}

scoreA := Score { Value: 10, IsRanked: true };
scoreB := Score { Value: 20, IsRanked: true };

// Forwards to 'Int', evaluates as 'scoreA.Value + 5'
newVal := scoreA + 5; 

// Forwards '+=' to the inline field
scoreA += 10; 

ProcessInt(val: Int) { }
ProcessInt(scoreA);       // ERROR: 'Score' is not implicitly an 'Int'
ProcessInt(scoreA.Value); // Valid
```
