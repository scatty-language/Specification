# Methods

Methods in Coggy are strictly categorized by the presence of the receiver parameter (`@`). They support expression-bodied syntax, explicit static dispatch, and fluent chaining.

```EBNF
MethodDecl      ::= Identifier "(" [Receiver] [ParameterList] ")" ["throws"] ["->" ReturnType] Body
Receiver        ::= ["mut"] "@" ["&"] [StateIdentifier] ["->" StateIdentifier] [","]
ParameterList   ::= Parameter ("," Parameter)*
Parameter       ::= ["mut"] Identifier ":" Type ["=" Expression]
ReturnType      ::= Type | "@" TypeModifier*
Body            ::= Block | "=>" Expression ";"
```

* **Implicit Void**: Explicitly declaring a void return type (e.g., `-> void`) is strictly FORBIDDEN. If a method does not return a value, the `-> ReturnType` clause MUST be completely omitted.
* **Expression Body**: Methods composed of a single expression MAY use the `=>` syntax instead of a block `{}`.
* **Default Parameters**: Methods MAY declare parameters with default values using the `=` syntax. Parameters with default values MUST appear after all parameters without default values.

## Instance Methods

An instance method operates on a specific instance of Data. It MUST declare `@` as its first parameter.

The receiver strictly dictates the ownership and mutability semantics of the instance.

* **By-Value (`@`)**: The method takes ownership of the instance. The instance is moved into the method and acts as an immutable local binding. It CANNOT be reassigned.
* **Mutable By-Value (`mut @`)**: The method takes ownership of the instance. The local binding is mutable, allowing field mutations and full reassignment of the `@` binding to a new instance of the same type.
* **Immutable View (`@&`)**: The method borrows the instance immutably. Ownership is not transferred. Fields CANNOT be mutated.
* **Mutable View (`mut @&`)**: The method borrows the instance mutably. Fields MAY be mutated, and the `@&` pointer itself MAY be reassigned locally to point to another instance.

```C#
Vector 
{
    X: int;
    Y: int;

    New(x: int, y: int) -> @
    {
        return Vector { x, y };
    }

    // @& borrows self immutably. 'other' is moved
    Sum(@&, other: Vector) -> Vector 
    {
        return New(@.X + other.X, @.Y + other.Y);
    }

    // Expression body borrowing self
    Print(@&) => Console.WriteLine($"{@.X, @.Y}");

    // Consumes self, allows replacement
    ConsumeAndReplace(mut @, other: Vector) 
    {
        @ = other; // Valid: @ is mutable by-value
    }
}
```

### Fluent Methods

A fluent method is an instance method that explicitly declares its return type as the exact same signature as its receiver (e.g., `-> @` or `-> @&`).

* **Implicit Return**: The compiler MUST AUTOMATICALLY injects the return of the instance (or pointer) at the end of the execution path.
* **Explicit Return Prohibition**: Manually writing `return @;` in a fluent method is redundant and explicitly FORBIDDEN.
* **Substitution**: If the receiver is mutable (`mut @` or `mut @&`), the developer MAY reassign `@` to a new instance (or pointer) within the method body. The fluent return will implicitly return the newly assigned instance.

```C#
Vector 
{
    X: int;
    Y: int;
        
    New(x: int, y: int) -> @
    {
        return Vector { x, y };
    }

    // Fluent method with mutable view borrowing
    Add(mut @&, other: Vector) -> @& 
    {
        @.X += other.X;
        @.Y += other.Y;
        
        // implicitly returns @&
    }

    // Fluent method with by-value ownership and substitution
    Replace(mut @, other: Vector) -> @
    {
        @ = other; // Substitutes the local instance entirely
        
        // implicitly returns the new @
    }
}

// Fluent chaining at call site
mut zero := Vector.New(0, 0);

zero
    .Add(Vector.New(1, 1))
    .Add(Vector.New(1, 1));
```

## Static Methods

Methods declared WITHOUT the `@` receiver parameter are static.

* **No Instance State**: They CANNOT access instance fields.
* **Invocation**: They MUST be invoked using the type name as a namespace (`DataName.MethodName()`).
* **Unqualified Call Inside Type**: When calling a static method from within the lexical scope of the same type, the type name MAY be omitted. An unqualified call resolves to the static method on the containing type.
* **Return Shorthand**: Static methods MAY declare their return type as `-> @`. In this context `@` is a shorthand for the current type in which the method is declared (the containing data type).

```C#
Vector 
{
    // Static method
    New(x: int, y: int) -> Vector 
    {
        return Vector { x, y };
    }

    // Static method with shorthand for the current type
    NewAnother(x: int, y: int) -> @
    {
        return Vector { x, y };
    }

    // Static method with unqualified static method call
    CreateZero() -> @
    {
        return New(0, 0);
    }
}

zero := Vector.New(0, 0);
```
