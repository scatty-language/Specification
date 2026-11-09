# Initialization

* MUST NOT implicitly zero-initialize memory.
* MUST guarantee memory initialization via exhaustive structural initialization.

## Positional and Named Assignment

* MAY mix `named_field_init` and `positional_field_init` within `struct_init_body`.
* MUST bind `named_field_init` explicitly to the specified field `Identifier`.
* MUST bind `positional_field_init` to the first structurally declared field not yet assigned within the `struct_init_body`.
* MUST dynamically shift the expected type order for subsequent positional arguments when a `named_field_init` is evaluated.

```scatty
MyData(MyInt: Int, MyFloat: Float, MyBool: Bool);

data1 := MyData { 0, 0.0, true };

data2 := MyData { MyFloat: 0.0, 0, true };
```

## Exhaustiveness and Default Values

* MUST evaluate and assign a value to every field defining the type's memory layout during `struct_init_body` instantiation.
* MAY explicitly assign a field's value via `positional_field_init` or via `named_field_init` within the `struct_init_body`.
* MUST implicitly evaluate and assign the declared default value for any field omitted from the `struct_init_body`.
* MUST override the declared default expression if a field is explicitly assigned.
* MUST NOT omit a field that lacks a declared default value.

## Memory Initialization Modifiers

* MAY explicitly bypass initialization exhaustiveness checks using the `zeroed` or `noinit` modifiers.
* MUST zero-initialize the entire allocated memory of the instance prior to executing the `struct_init_body` when the `zeroed` modifier is applied.
* MUST initialize the discriminant tag to the tag value of the first declared variant when `zeroed` is applied to an unnarrowed tagged `Data` type (`Type { ... } zeroed`).
* MUST initialize the discriminant tag to the specified variant's tag value when `zeroed` is applied to a variant constructor (`Type::Variant { ... } zeroed`).
* MAY omit any field, including those without default values, from the `struct_init_body` when `zeroed` or `noinit` is applied.
* MUST confine the use of the `zeroed` modifier to within an `unsafe` block or `unsafe` method if any field omitted from the initialization (or any nested element within the recursive memory layout of its shared fields or active variant payload) is:
  * A non-optional Owning Type (`~Disposable`),
  * A non-optional function pointer (`fn`),
  * A guarded type (`Type^Guard`) whose zero-initialized value does not satisfy the guard's `must` invariant at compile time,
  * A nested tagged `Data` type whose first declared variant has a non-zero discriminant tag (`!= 0`), or
  * A nested variant subtype (`Data::Variant`) whose discriminant tag is non-zero (`!= 0`).
* MAY apply the `zeroed` modifier in safe code when all omitted fields in the target `Data` type's recursive memory layout are unguarded Trivial Types (with discriminant tag `0` for nested tagged types), Option types (`T?`), weak references (`T weak`), or guarded types (`Type^Guard`) whose `must` invariant evaluates to `true` for a zero-initialized value at compile time.
* MUST leave the memory of any omitted field entirely uninitialized (containing arbitrary garbage data) when the `noinit` modifier is applied.
* MUST confine the use of the `noinit` modifier to within an `unsafe` block or `unsafe` method.

```scatty
^NonNegative = Int must @ >= 0;
^Positive    = Int must @ > 0;

PacketStats 
{
    Sequence: Int;
    BytesSent: Int^NonNegative;
    OptionalTag: String?;
}

HttpResponse 
{
    StatusCode: Int;
    RequestUrl: String;
    Callback: fn(Int);
}

Account 
{
    Id: Int^Positive;
}

// Valid in safe code: all fields are Trivial, Option ('None' == 0x0), or satisfied by 0 ('0 >= 0')
stats := PacketStats { } zeroed;

// ERROR: 'HttpResponse' contains non-optional Owning Type ('String') and 'fn' pointer
badResponse := HttpResponse { } zeroed;

// ERROR: 'Account.Id' has guard '^Positive' which evaluates to false for 0 at compile time
badAccount := Account { } zeroed;

unsafe 
{
    rawResponse := HttpResponse { } zeroed;
    fastResponse := HttpResponse { } noinit;
}
```

## Variant Instantiation

* MUST initialize variant payload fields via `variant_payload_init_body`, adhering to Method Invocation rules.
* MUST initialize shared state fields via `struct_init_body`, adhering to Structural Initialization rules, when constructing a new `Data` instance.
* MAY omit the `struct_init_body` during new instance construction if and only if the `Data` type declares no shared fields or if all shared fields possess default values.
* MUST omit the `variant_payload_init_body` if the variant declares no payload fields.
* MUST NOT assign a payload field via named assignment if its declaration omits an identifier.

```scatty
HttpResponse
{
    ::Success;
    ::Error(Code: Int, Message: String = "Fatal");

    Timestamp: Int;
}

response1 := HttpResponse::Success { Timestamp: 12345 };

response2 := HttpResponse::Error(Message: "Not Found", 404) { Timestamp: 12345 };

Option<T> 
{
    ::None;
    ::Some(T);
}

option1 := Option::None; 

option2 := Option::Some(42); 
```

### Active Variant Switching

* MAY switch the active variant of an already initialized mutable variable or field of a base `Data` type in-place using the `=::` operator (`target =:: Variant` or `target =:: Variant(args)`).
* MUST NOT include a leading dot (`.`), a type qualifier (`Type::`), or a `struct_init_body` (`{ ... }`) on the right-hand side of the `=::` operator.
* MUST update only the active discriminant tag and the variant payload during an in-place variant switch (`=::`).
* MUST preserve the current values of all shared fields on the target instance during an in-place variant switch (`=::`) without re-evaluating or resetting shared field default values.
* MUST NOT apply the `=::` operator to a variable, field, or parameter that is statically typed or flow-narrowed to a specific variant subtype (`Data::Variant`).
* MUST replace the entire `Data` instance—including all shared fields—when assigning a variant constructor using the standard assignment operator (`=`).

```scatty
Session 
{
    ::Disconnected;
    ::Connected(SocketId: Int);
    ::Failed(Reason: String);

    Host: String;
    Retries: Int = 0;
}

session := Session::Disconnected { Host: "127.0.0.1", Retries: 3 };

// In-place variant switch via '=::':
// Updates active variant to '::Connected(42)' while preserving 'Host' ("127.0.0.1") and 'Retries' (3)
session =:: Connected(42);

// Full instance replacement via '=':
// Replaces the entire instance; omitted 'Retries' resets to its default value (0)
session = Session::Failed("Timeout") { Host: "10.0.0.1" };
```

## Visibility and Forced Encapsulation

* MUST enforce identifier visibility rules during structural initialization.
* MUST NOT explicitly assign a value to a restricted (private or internal) field from outside its permitted lexical scope.
* MUST NOT instantiate a Data type via `struct_init_body` from outside its permitted scope if it contains a restricted field lacking a default value.

```scatty
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

```scatty
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
