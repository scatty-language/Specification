# Embedded Fields

* MAY declare an embedded field within a `Data` type by placing the `embed` modifier immediately after the field's type (`Identifier: Type embed`).
* MUST automatically expose public and internal members of the embedded field's type directly on the enclosing `Data` type.
* MUST implicitly forward method and property accesses from the parent instance to the embedded field.
* MUST automatically pass the embedded field by reference if the forwarded method requires a `ref` receiver.
* MUST permit direct access to the embedded field via its declared identifier subject to standard visibility rules.
* MUST NOT combine the `embed` modifier with `static`, `astatic`, or `gstatic` on the same field.

## Embedded Variant Fields

* MUST NOT declare more than one embedded variant field within a single `Data` declaration.
* MUST transparently forward state evaluations (`is`, `as`, `as?`, `as!`, `assume`, `match`) applied to the parent instance directly to the underlying active tag of the embedded variant field.
* MUST transparently forward all methods and properties of the embedded variant field's specified base type to the parent instance.
* MUST explicitly assign the embedded variant field using its identifier during structural instantiation of the parent type unless initialized via a forwarded variant constructor.

```scatty
StatusMessage
{
    Message: String embed = "Wow";
    ::Success;
    ::Error(Code: Int);
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

// ERROR: Calls 'wrapper.Add' if unambiguous
wrapper.Add(1);
```

## Operator Forwarding and Primitive Types

* MUST delegate arithmetic operators (`+`, `-`, `*`, `/`, `%` and compound assignments `+=`, `-=`, `*=`, `/=`, `%=`) to an embedded field (including an embedded primitive field) if the parent `Data` type declares zero `math` fields and the embedded field's type supports arithmetic operators.
* MUST require both operands of a delegated binary arithmetic operator or compound assignment to evaluate to the exact same parent `Data` type.
* MUST evaluate a delegated binary arithmetic operator by applying the operator to the corresponding embedded fields of both operands, copying all remaining non-delegated fields from the left-hand operand, and yielding a new instance of the parent `Data` type.
* MUST evaluate a delegated compound assignment operator (`+=`, `-=`, `*=`, `/=`, `%=`) by applying the operator in-place to the left-hand operand's embedded field using the right-hand operand's embedded field, leaving all other fields of the left-hand operand unchanged.
* MUST shadow and override embedded field operator delegation if the parent `Data` type explicitly declares any `math` fields.
* MUST NOT implicitly typecast or subtype the parent `Data` type to the type of its embedded field.
* MUST NOT mix the parent `Data` type and the embedded field's type as operands of a delegated operator.

```scatty
Score 
{
    Value: Int embed;
    IsRanked: Bool;
}

scoreA := Score { Value: 10, IsRanked: true };
scoreB := Score { Value: 20, IsRanked: false };

// Valid: evaluates 'scoreA.Value + scoreB.Value' and preserves 'scoreA.IsRanked' (true)
// Result is 'Score { Value: 30, IsRanked: true }'
totalScore: Score = scoreA + scoreB;

// ERROR: both operands MUST have the exact same type ('Score'); cannot mix 'Score' and 'Int'
badScore := scoreA + 5;

ProcessInt(val: Int) { }

// ERROR: 'Score' does not implicitly convert to 'Int'
ProcessInt(scoreA);

// Valid: explicit access to the embedded field
ProcessInt(scoreA.Value);
```
