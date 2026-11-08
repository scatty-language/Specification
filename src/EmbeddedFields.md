# Embedded Fields

* MAY declare an embedded field within a `Data` type by placing the `embed` modifier immediately after the field's type (`Identifier: Type embed`).
* MUST automatically expose public and internal members of the embedded field's type directly on the enclosing `Data` type.
* MUST implicitly forward method and property accesses from the parent instance to the embedded field.
* MUST automatically pass the embedded field by reference if the forwarded method requires a `ref` receiver.
* MUST permit direct access to the embedded field via its declared identifier subject to standard visibility rules.
* MUST NOT combine the `embed` modifier with `static`, `astatic`, or `gstatic` on the same field.

## Embedded Variant Fields

* MUST NOT declare more than one embedded variant field within a single `Data` declaration.
* MUST transparently forward state evaluations (`is`, `as`, `as?`, `as!`, `unas`, `match`) applied to the parent instance directly to the underlying active tag of the embedded variant field.
* MUST transparently forward all methods and properties of the embedded variant field's specified base type to the parent instance.
* MUST explicitly assign the embedded variant field using its identifier during structural instantiation of the parent type unless initialized via a forwarded variant constructor.

```scatty
StatusMessage
{
    Message: String embed = "Wow";
    ::Success;
    ::Error(code: Int);
}

Response 
{
    Status: StatusMessage embed;
    Timestamp: Int;
}

res := Response::Success { Timestamp: 12345 };

may res is .Error(code) 
{
    Console.WriteLine(code);
}

may res.Contains("Wow") 
{
    Console.WriteLine("String method forwarded!");
}
```

## Shadowing and Collisions

* MUST explicitly override an embedded field's exposed method or property if the parent type declares a member with an identical name.
* MUST reject compilation if the parent type explicitly declares a structural data field with the identical name as a structural data field inside the embedded type.
* MUST reject compilation on ambiguous access if multiple embedded fields within the same `Data` type expose a member with the identical name.
* MUST resolve ambiguity by explicitly declaring a shadowing method or property on the parent type that manually delegates to a specific embedded field.

```scatty
Wrapper<T> 
{
    List: [T] embed;
    ListB: [T] embed;

    WowCount: Int;

    // Overrides 'Pop' method
    Pop(@) 
    {
        Console.WriteLine("Intercepted Pop");
    }

    Capacity: Int => @.ListB.Capacity;
}

wrapper := Wrapper<Int> { List: [Int].New(), WowCount: 0, ListB: [Int].New() };

wrapper.List.Add(1);

// Calls 'wrapper.List.Add' if unambiguous
```

## Operator Forwarding and Primitive Types

* MUST transparently forward operators from an embedded primitive field to the parent type, evaluating them as structural method calls.
* MUST evaluate forwarded operators on the embedded field if the parent type does not explicitly shadow the operator.
* MUST NOT implicitly typecast or subtype the parent `Data` type to the type of its embedded field.

```scatty
Score 
{
    Value: Int embed;
    IsRanked: Bool;
}

scoreA := Score { Value: 10, IsRanked: true };
scoreB := Score { Value: 20, IsRanked: true };

// Evaluates as 'scoreA.Value + 5'
newVal := scoreA + 5; 

ProcessInt(val: Int) { }

ProcessInt(scoreA.Value);
```
