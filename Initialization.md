# Initialization

To guarantee memory safety and zero runtime overhead, Coggy DOES NOT zero-initialize memory implicitly. Uninitialized memory is prevented at compile time via exhaustive structural initialization.

```EBNF
StructInitExpr  ::= TypePath "{" [FieldInitList] "}"
FieldInitList   ::= FieldInit ("," FieldInit)* ","?
FieldInit       ::= [Identifier ":"] Expression
TypePath        ::= Identifier | "."
```

## Exhaustiveness and Default Values

When instantiating a `Data` type using the `{}` block syntax, every field defining the type's memory layout MUST receive a evaluated value.

* **Explicit Initialization**: A field's value MAY be explicitly provided inside the `{}` block, either positionally or by name.
* **Implicit Fallback**: If a field is completely omitted from the `{}` block, the field MUST have a default value (`= Expression`) declared in the `Data` type definition. The compiler implicitly evaluates and assigns this default value.
* **Override**: If an explicitly provided field also possesses a default value in its declaration, the explicit value OVERRIDES the default expression. The default expression is NOT evaluated.
* **Compilation Error**: Omitting a field that lacks a default declaration is FORBIDDEN.

## Positional and Named Assignment

Coggy supports both explicitly named (memberwise) and positional field assignments within the `{}` block. These styles CAN be freely mixed.

* **Named Assignment (`Identifier: Expression`)**: Binds the evaluated expression explicitly to the specified field name.
* **Positional Assignment (`Expression`)**: If the field identifier is omitted, the expression is bound to the first structurally declared field in the `Data` type that has NOT YET been assigned within the current `{}` block.
* **Dynamic Type Order**: Using a named assignment dynamically shifts the expected type order for any subsequent positional arguments.

```C#
// Declaration order: MyInt, MyFloat, MyBool
MyData(MyInt: Int, MyFloat: Float, MyBool: Bool);

// Fills in declaration order. Required type order: 'Int', 'Float', 'Bool'
data1 := MyData { 0, 0.0, true };

// 'MyFloat' is explicitly assigned
// The first positional '0' binds to the first unassigned field ('MyInt')
// The second positional 'true' binds to the next unassigned field ('MyBool')
// Required type order: 'Float', 'Int', 'Bool'
data2 := MyData { MyFloat: 0.0, 0, true };

// 'MyBool' and 'MyFloat' are assigned.
// The only remaining unassigned field is 'MyInt'
// Required type order: 'Bool', 'Float', 'Int'
data3 := MyData { MyBool: true, MyFloat: 0.0, 0 };
```

## Visibility and Forced Encapsulation

Structural initialization respects identifier visibility rules.

* **Direct Access Prohibition**: A developer MUST NOT explicitly assign a value to a restricted (private or internal) field from outside its permitted lexical scope.
* **Forced Encapsulation**: If a `Data` type contains a restricted field that DOES NOT have a default value, the type CANNOT be instantiated via `{}` from outside its permitted scope. The author of the type MAY provide a public method to permit external instantiation.

```C#
// PACKAGE: Core

Config 
{
    Timeout: Int;
    Retries: Int = 3;
}

Connection 
{
    Host: String;
    _socket: Int;      // Private, NO default value
    _flags: Int = 0;   // Private, WITH default value
    
    // Public factory method
    New(host: String) -> @ 
    {
        // Inside the allowed scope, we can access and initialize '_socket'
        // '_flags' is implicitly initialized to 0
        return Connection { Host: host, _socket: System.CreateSocket() };
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
        // 'Timeout' is explicit, 'Retries' implicitly takes the default (3)
        c1 := Config { Timeout: 5000 };
        
        // Explicit value overrides the default, 'Retries' is now 5
        c2 := Config { Timeout: 5000, Retries: 5 };

        // ERROR: Exhaustiveness violation. 'Timeout' has no default and is omitted
        badConfig := Config { };

        // ERROR: Visibility violation, cannot explicitly initialize private '_socket'
        conn1 := Connection { Host: "localhost", _socket: 1 };

        // ERROR: Forced encapsulation, '_socket' is uninitialized and inaccessible
        conn2 := Connection { Host: "localhost" };

        // Instantiation via the permitted method
        conn3 := Connection.New("localhost");
    }
}
```
