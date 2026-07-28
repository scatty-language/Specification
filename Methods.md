# Methods

Methods in Coggy are categorized by the presence of the receiver parameter.

```EBNF
MethodDecl      ::= Identifier "(" [Receiver] [ParameterList] ")" MethodModifiers* ["->" ReturnType] (Body | ";")
Receiver        ::= ["ref"] "@" [","]
ParameterList   ::= Parameter ("," Parameter)*
Parameter       ::= ["ref"] LocalId ":" Type ["=" Expression]
ReturnType      ::= Type | "@"
Body            ::= Block | "=>" Expression ";"
MethodModifiers ::= "throws" | "unsafe"
```

* **Implicit Void**: Explicitly declaring a void return type (e.g., `-> void`) is FORBIDDEN. If a method does not return a value, the `-> ReturnType` clause MUST be omitted.
* **Expression Body**: Methods composed of a single expression MAY use the `=>` syntax instead of a block `{}`.
* **Default Parameters**: Methods MAY declare parameters with default values using the `=` syntax. Parameters with default values MUST appear after all parameters without default values.

## Instance Methods

An instance method operates on a specific instance of Data.

* **Receiver Position**: It MUST be declared as the first parameter once.
* **By-Value (`@`)**: The method receives a logically independent copy of the instance. The method MAY mutate `@` locally, but these mutations are isolated and DO NOT affect the caller's original instance.
* **In-Place Mutation (`ref @`)**: The method receives a direct reference to the caller's instance. Any mutations applied to `@` modify the original instance.
* **No Reassignment**: Reassigning the `@` itself (e.g., `@ = Data.New();`) is FORBIDDEN.

```C#
Vector
{
    X: Int;
    Y: Int;

    New(x: Int, y: Int) -> @ => Vector { X: x, Y: y };

    // @ is passed by value
    // 'other' is passed by value
    // The original instance remains unchanged
    Sum(@, other: Vector) -> Vector 
    {
        return Vector.New(@.X + other.X, @.Y + other.Y);
    }

    // Expression body, reads the local value
    Print(@) => Console.WriteLine($"{@.X}, {@.Y}");

    // Mutates the caller's original instance in-place
    Translate(ref @, dx: Int, dy: Int) 
    {
        @.X += dx;
        @.Y += dy;
    }
}
```

### Fluent Methods

A fluent method is an instance method that explicitly declares its return type as `@`.

* **Implicit Return**: The compiler MUST inject the return of the instance at the end of the execution path.
* **Explicit Return Prohibition**: Manually writing `return @;` in a fluent method is FORBIDDEN.
* **Builder Pattern**: If the receiver is `@`, the method mutates its local copy and implicitly returns it.
* **Chaining Mutations**: If the receiver is `ref @`, the method mutates the original instance and implicitly returns a copy of the mutated state.

```C#
Vector 
{
    X: Int;
    Y: Int;
        
    New(x: Int, y: Int) -> @ => Vector { X: x, Y: y };

    // Builder pattern: operates on a value copy and returns the new copy
    WithX(@, x: Int) -> @ 
    {
        @.X = x;
        // implicitly returns '@'
    }

    // Fluent in-place mutation
    Add(ref @, other: Vector) -> @ 
    {
        @.X += other.X;
        @.Y += other.Y;
        // implicitly returns '@'
    }
}

// Builder chaining, original 'zero' is untouched
zero := Vector.New(0, 0);
unit := zero.WithX(1).WithY(1); 

// In-place fluent chaining
pos := Vector.New(10, 10);
pos.Add(Vector.New(5, 0)).Add(Vector.New(0, 5)); 
// 'pos' is now { X: 15, Y: 15 }
```

## Static Methods

Methods declared WITHOUT the `@` receiver parameter are static.

* **No Instance State**: They CANNOT access instance fields.
* **Invocation**: They MUST be invoked using the type name as a namespace (`DataName.MethodName()`).
* **Unqualified Call Inside Type**: When calling a static method from within the lexical scope of the same type, the type name MAY be omitted. An unqualified call resolves to the static method on the containing type.
* **Return Shorthand**: Static methods MAY declare their return type as `-> @`. In this context, `@` is a syntactic shorthand for the current Data type in which the method is declared.

```C#
Vector 
{
    // Static method returning the explicitly named type
    CreateEmpty() -> Vector 
    {
        return Vector { X: 0, Y: 0 };
    }

    // Static method using the '@' shorthand for the return type
    New(x: Int, y: Int) -> @
    {
        return Vector { X: x, Y: y };
    }

    // Unqualified static method call
    CreateZero() -> @
    {
        return New(0, 0);
    }
}

zero := Vector.CreateZero();
```
