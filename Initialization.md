# Initialization

* MUST NOT implicitly zero-initialize memory.
* MUST guarantee memory initialization via exhaustive structural initialization.

## Positional and Named Assignment

* MAY mix `NamedFieldInit` and `PositionalFieldInit` within `StructInitBody`.
* MUST bind `NamedFieldInit` explicitly to the specified field `Identifier`.
* MUST bind `PositionalFieldInit` to the first structurally declared field not yet assigned within the `StructInitBody`.
* MUST dynamically shift the expected type order for subsequent positional arguments when a `NamedFieldInit` is evaluated.

```C#
MyData(MyInt: Int, MyFloat: Float, MyBool: Bool);

data1 := MyData { 0, 0.0, true };

data2 := MyData { MyFloat: 0.0, 0, true };
```

## Exhaustiveness and Default Values

* MUST evaluate and assign a value to every field defining the type's memory layout during `StructInitBody` instantiation.
* MAY explicitly assign a field's value via `PositionalFieldInit` or via `NamedFieldInit` within the `StructInitBody`.
* MUST implicitly evaluate and assign the declared default value for any field omitted from the `StructInitBody`.
* MUST override the declared default expression if a field is explicitly assigned.
* MUST NOT omit a field that lacks a declared default value.

## Memory Initialization Modifiers

* MAY explicitly bypass initialization exhaustiveness checks using the `zeroed` or `noinit` modifiers.
* MUST zero-initialize the entire allocated memory of the instance prior to executing the `StructInitBody` when the `zeroed` modifier is applied.
* MAY omit any field, including those without default values, from the `StructInitBody` when `zeroed` or `noinit` is applied.
* MUST leave the memory of any omitted field entirely uninitialized (containing arbitrary garbage data) when the `noinit` modifier is applied.
* MUST confine the use of the `noinit` modifier to within an `unsafe` block.

```C#
response := HttpResponse { } zeroed;

unsafe 
{
    fastResponse := HttpResponse { } noinit;
}
```

## Variant Instantiation

* MUST initialize variant payload fields via `VariantPayloadInitBody`, adhering to Method Invocation rules.
* MUST initialize shared state fields via `StructInitBody`, adhering to Structural Initialization rules, when constructing a new `Data` instance.
* MAY omit the `StructInitBody` during new instance construction if and only if the `Data` type declares no shared fields or if all shared fields possess default values.
* MUST omit the `VariantPayloadInitBody` if the variant declares no payload fields.
* MUST NOT assign a payload field via named assignment if its declaration omits an identifier.

```C#
HttpResponse
{
    ::Success;
    ::Error(code: Int, message: String = "Fatal");

    Timestamp: Int;
}

response1 := HttpResponse::Success { Timestamp: 12345 };

response2 := HttpResponse::Error(message: "Not Found", 404) { Timestamp: 12345 };

Option<T> 
{
    ::None;
    ::Some(T);
}

option1 := Option::None; 

option2 := Option::Some(42); 
```

### Active Variant Switching

* MAY assign a variant constructor expression without a `StructInitBody` (`Type::Variant(...)` or `.Variant(...)`) to an already initialized mutable variable or field of the enclosing base `Data` type.
* MUST treat a variant assignment lacking a `StructInitBody` on an existing instance as an in-place variant switch.
* MUST update only the active discriminant tag and the variant payload during an in-place variant switch.
* MUST preserve the current values of all shared fields on the target instance during an in-place variant switch without re-evaluating or resetting shared field default values.
* MUST replace the entire `Data` instance—including all shared fields—if a `StructInitBody` (`{ ... }`) is explicitly provided on the assigned variant expression.

```C#
Session 
{
    ::Disconnected;
    ::Connected(SocketId: Int);
    ::Failed(Reason: String);

    Host: String;
    Retries: Int = 0;
}

session := Session::Disconnected { Host: "127.0.0.1", Retries: 3 };

// In-place variant switch (omits '{}'):
// Updates active variant to '::Connected(42)' while preserving 'Host' ("127.0.0.1") and 'Retries' (3)
session = .Connected(42);

// Full instance replacement (includes '{}'):
// Replaces the entire instance; omitted 'Retries' resets to its default value (0)
session = Session::Failed("Timeout") { Host: "10.0.0.1" };
```

## Visibility and Forced Encapsulation

* MUST enforce identifier visibility rules during structural initialization.
* MUST NOT explicitly assign a value to a restricted (private or internal) field from outside its permitted lexical scope.
* MUST NOT instantiate a Data type via `StructInitBody` from outside its permitted scope if it contains a restricted field lacking a default value.

```C#
// PACKAGE: Core
Connection 
{
    Host: String;
    _socket: Int;
    _flags: Int = 0;
    
    New(host: String) -> @ 
    {
        return Connection { Host: host, _socket: Connector.CreateSocket() };
    }
}
```

```C#
// PACKAGE: App
use Core;

App 
{
    Run() 
    {
        conn3 := Connection.New("localhost");
    }
}
```
