# Inline Fields

```EBNF
InlineFieldDecl ::= "@" FieldDecl | VariantFieldDecl
```

* MUST automatically expose public and internal members of the inline field's type directly on the enclosing `Data` type.
* MUST implicitly forward method and property accesses from the parent instance to the inline field.
* MUST automatically pass the inline field by reference if the forwarded method requires a `ref` receiver.
* MUST permit direct access to the inline field via its declared identifier subject to standard visibility rules.

## Inline Variant Fields

* MUST NOT declare more than one inline variant field within a single `Data` declaration.
* MUST transparently forward state evaluations (`is`, `as`, `match`) applied to the parent instance directly to the underlying active tag of the inline variant field.
* MUST transparently forward all methods and properties of the inline variant field's specified base type to the parent instance.
* MUST explicitly assign the variant field using its identifier during structural instantiation of the parent type.

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

if res.Contains("Wow") 
{
    Console.WriteLine("String method forwarded!");
}
```

## Shadowing and Collisions

* MUST explicitly override an inline field's exposed method or property if the parent type declares a member with an identical name.
* MUST reject compilation if the parent type explicitly declares a structural data field with the identical name as a structural data field inside the inline type.
* MUST reject compilation on ambiguous access if multiple inline fields within the same `Data` type expose a member with the identical name.
* MUST resolve ambiguity by explicitly declaring a shadowing method or property on the parent type that manually delegates to a specific inline field.

```C#
Wrapper<T> 
{
    @List: [+ T];
    @ListB: [+ T];

    WowCount: Int;

    // Overrides 'Pop' method
    Pop(@) 
    {
        Console.WriteLine("Intercepted Pop");
    }

    Capacity: Int => @.ListB.Capacity;
}

wrapper := Wrapper<Int> { List: [+ Int].New(), WowCount: 0, ListB: [+ Int].New() };

wrapper.List.Add(1);

// Calls 'wrapper.List.Add'
wrapper.Add(0);
```

## Operator Forwarding and Primitive Types

* MUST transparently forward operators from an inline primitive field to the parent type, evaluating them as structural method calls.
* MUST evaluate forwarded operators on the inline field if the parent type does not explicitly shadow the operator.
* MUST NOT implicitly typecast or subtype the parent `Data` type to the type of its inline field.

```C#
Score 
{
    @Value: Int;
    IsRanked: Bool;
}

scoreA := Score { Value: 10, IsRanked: true };
scoreB := Score { Value: 20, IsRanked: true };

// Evaluates as 'scoreA.Value + 5'
newVal := scoreA + 5; 

ProcessInt(val: Int)
{

}

ProcessInt(scoreA.Value);
```
