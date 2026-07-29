# Generics

Generics allow data structures and methods to operate on interfaces without sacrificing memory safety or performance. Coggy implements generics via strict monomorphization.

## Type Parameters and Constraints

Type parameters are declared using angle brackets `<T>`. Constraints define the capabilities a type must possess, using Interfaces or Refinement Types.

```EBNF
GenericParams  ::= "<" TypeParameter ("," TypeParameter)* ">" ["where" Expression]
TypeParameter  ::= Identifier [":" ConstraintList]
ConstraintList ::= TypeIdentifier ("," TypeIdentifier)*
```

* **Constraints as Contracts**: A type parameter `T: ~Printable` guarantees that `T` inherits/implements the `~Printable` interface.
* **Multiple Constraints**: Multiple constraints are separated by commas (`T: ~Printable, ~Copyable`). The type MUST satisfy all constraints simultaneously.

```C#
// Generic data with an unconstrained type
Cell<T> 
{
    Value: T;
}

// Generic data constrained to types that implement '~Printable'
PrinterData<T: ~Printable> 
{
    Item: T;

    PrintItem(@) => @.Item.Print();
}
```

### Structural Constraints

Developers MAY enforce structural requirements on generic type parameters using a `where` clause paired with compile-time reflection (`meta<T>`).

* **Boolean Requirement**: The `where` clause MUST evaluate to a pure, compile-time boolean expression.
* **Monomorphization Validation**: During generic monomorphization, the compiler statically evaluates the expression for the specific type `T`. If it resolves to `false`, the compiler MUST rejects the type and emits a compilation error.
* **Composition**: Structural constraints (`where`) MAY be combined seamlessly with standard interface behavioral constraints (`:`).

```C#
// Constrains 'T' to be a standard Data type
Repository<T> where meta<T>.Kind == .Data 
{
    Items: List<T>;
}

// Requires 'T' to implement an interface AND satisfy a structural predicate
Process<T: ~Printable>(item: T) where meta<T>.HasIdentity 
{
    item.Print();
}
```

## Static Dispatch

When a method or Data uses a generic parameter `<T: ~Contract>`, the compiler makes a monomorphization of the code for each specific type used.

* **Devirtualization**: Any call to a virtual method (e.g., `item.Print()`) on a generic parameter `T` is resolved at compile time as a direct, static method call.

```C#
// If called with 'Player', a specific Player version is generated
ProcessItem<T: ~Entity, ~Printable>(item: T) 
{
    item.Update(); // Direct static call
    item.Print();  // Direct static call
}
```

## Generic Methods

Methods within standard Data or modules MAY declare their own generic parameters. Generic parameters of a method MUST NOT shadow the generic parameters of the enclosing `Data`.

```C#
Processor 
{
    // Generic method on a non-generic Data type
    Process<T: ~Printable>(item: T) 
    {
        item.Print();
    }
}
```

## Generics and Compile-Time Reflection

Compile-time reflection is fully supported inside generic methods and `Data` types.

* **Static Baking**: When a generic method `<T>` is monomorphized into a concrete type, the compiler bakes the specific `meta<T>` constants directly into that specific function instance.
* **Loop Unrolling**: If a `for` loop iterates over a `meta<T>` array, the compiler MAY fully unroll the loop, resulting in direct sequential execution.

```C#
Serializer 
{
    Serialize<T>(item: T) 
    {
        // For each distinct 'T', this array is a hardcoded constant
        fields := meta<T>.Fields; 

        for field in fields 
        {
            if field.IsIdentity 
            {
                Console.WriteLine($"Identity Field: {field.Name}");
            }
        }
    }
}
```
