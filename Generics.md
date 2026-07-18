# Generics

Generics allow data structures and methods to operate on abstracted types without sacrificing memory safety or performance. Coggy implements generics via strict monomorphization.

## Type Parameters and Constraints

Type parameters are declared using angle brackets `<T>`. Constraints define the capabilities a type must possess, strictly using Abstract Data (`~Data`) or Refinement Types.

```EBNF
GenericParams  ::= "<" TypeParameter ("," TypeParameter)* ">"
TypeParameter  ::= Identifier [":" ConstraintList]
ConstraintList ::= TypeIdentifier ("," TypeIdentifier)*
```

* **Constraints as Contracts**: A type parameter `T: ~IPrintable` guarantees that `T` inherits/implements the `~IPrintable` abstract data.
* **Multiple Constraints**: Multiple constraints are separated by commas (`T: ~IPrintable, ~ICopy`). The type MUST satisfy all constraints simultaneously.

```C#
// Generic data with an unconstrained type
Cell<T> 
{
    Value: T;
}

// Generic data constrained to types that implement '~IPrintable'
PrinterData<T: ~IPrintable> 
{
    Item: T;

    PrintItem(@) => @.Item.Print();
}
```

## Static Dispatch

When a method or Data uses a generic parameter `<T: ~Contract>`, the compiler strictly makes a monomorphization of the code for each specific type used.

* **Devirtualization**: Any call to a virtual method (e.g., `item.Print()`) on a generic parameter `T` is strictly resolved at compile time as a direct, static method call.

```C#
// Monomorphized method
// If called with 'Player', a specific Player version is generated
ProcessItem<T: ~Entity, ~IPrintable>(item: T) 
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
    Process<T: ~IPrintable>(item: T) 
    {
        item.Print();
    }
}
```
