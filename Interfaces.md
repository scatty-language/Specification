# Interfaces

Interfaces define stateless behavioral contracts. They allow multiple `Data` types to guarantee the implementation of specific methods without enforcing a memory layout.

```EBNF
InterfaceDecl     ::= "~" Identifier "{" InterfaceMember* "}"
InterfaceMember   ::= InterfacePropDecl | MethodDecl | RefinementDecl
InterfacePropDecl ::= Identifier ":" Type ( ";" | "=>" Expression ";" | Block )
```

* **Zero Memory Footprint**: An interface MUST NOT contain any state fields. Interfaces MUST occupy 0 bytes.
* **Multiple Implementation**: A `Data` type MAY inherit an UNLIMITED number of interfaces.
* **No Instantiation**: Interfaces MUST NOT be directly instantiated via the initializer syntax (`~Interface {}`).

## Method Contracts and Default Implementations

* **Contract Methods**: Declared without a body (ending with `;`). Any `Data` type implementing the interface MUST provide a concrete method with the exact same signature.
* **Default Methods**: Declared with a body (`{ ... }`). The interface provides a shared default behavior. The implementing `Data` type implicitly receives this method.
* **Shadowing**: A `Data` type MAY shadow a default interface method by explicitly declaring a method with the exact same signature.
* **Implicit Resolution**: Interface satisfaction is signature-based. The compiler automatically maps the `Data` method to the interface contract.

```C#
~Comparable 
{
    // Contract Method: MUST be implemented by Data
    Compare(@, other: @) -> Int;

    // Default Method
    IsEqual(@, other: @) -> Bool 
    {
        return @.Compare(other) == 0;
    }
}

Score : ~Comparable 
{
    Value: Int;

    Compare(@, other: @) -> Int 
    {
        if @.Value > other.Value => return 1;
        if @.Value < other.Value => return -1;
        return 0;
    }

    // 'IsEqual' is implicitly available without redeclaration
}
```

## Property Contracts and Satisfaction

Interfaces MAY declare properties to enforce data access contracts.

* **Contract Properties**: Declared with a name and type, ending with `;`. The interface dictates that the implementing type MUST provide read access to this value.
* **Default Properties**: Declared with a body (`=>` or `{ ... }`). The interface provides a shared computed property. The implementing `Data` type implicitly receives it and MAY shadow it.
* **Field-to-Property Satisfaction**: To satisfy a contract property, the implementing `Data` type MAY explicitly declare a computed property (`Property: Type => ...`) OR simply declare a standard state field with the exact same name and type (`Property: Type;`).
* **Dynamic Dispatch (VTable)**: If an interface property is accessed via a dynamic reference (`ref ~Interface`), the compiler MUST implicitly generate a transparent getter function in the vtable to resolve the field or property dynamically.

```C#
~Vectorable 
{
    // Contract properties
    X: Int; 
    Y: Int;

    // Default computed property using '@' to access instance members
    DoubleX: Int => @.X * 2; 

    // Default block-bodied property
    LengthSquared: Int  
    {
        return (@.X * @.X) + (@.Y * @.Y);
    }
}

Entity : ~Vectorable 
{
    // Standard fields implicitly satisfy the 'X' and 'Y' property contracts
    X: Int; 
    Y: Int;
    
    // 'DoubleX' and 'LengthSquared' are implicitly inherited
}
```

## The `@` Receiver and Self-Resolution

Within an Interface declaration, the `@` symbol represents the Concrete Implementing Type, NOT the interface itself.

* When a method declares `-> @`, it dictates that the implementing type MUST return a new instance of its exact concrete type.
* If a method needs to interact with ANY type that implements the interface, it MUST explicitly declare the parameter using the interface type via reference (`ref ~Interface`) or standard containers.

```C#
~Equatable 
{
    // '@' is the concrete type (e.g., 'Button' comparing to another 'Button')
    Equals(@, other: @) -> Bool;

    // 'ref ~Equatable' is ANY type implementing the interface
    IsSameType(@, ref other: ~Equatable) -> Bool;
}

~Copyable
{
    // Requires returning a new instance of the concrete type
    Copy(@) -> @;
}
```

## Object Safety and Dispatch Restrictions

Interfaces can be utilized in two ways: via Generics or via dynamic References/Boxes.

1. **Static Dispatch (Generics)**: ANY method defined in an Interface CAN be called on a generic parameter `T: ~Interface`. The compiler MUST monomorphize the code, knowing the exact physical layout of the concrete type at compile time.
2. **Dynamic Dispatch (VTable)**: When interacting with an instance via a dynamic reference (`ref item: ~Interface`) or a container, the exact concrete type and its memory size are erased at compile time.
   * **Return Types**: Methods that return the concrete type (`-> @`) are FORBIDDEN to be called dynamically.
   * **Object Safety**: Methods that take `@` or `ref @` and do not return `@` are fully Object Safe and CAN be called dynamically.

```C#
// Static Generic Context
// Fully Object Safe, 'T' is known at compile time
Duplicate<T: ~Copyable>(item: T) 
{
    newItem := item.Copy();
}

// Dynamic VTable Context
ProcessDynamic(ref item: ~Copyable) 
{
    // ERROR: 'Copy' returns '@' (concrete type)
    // The exact size of '@' is unknown for a dynamic reference
    val := item.Copy(); 
}
```
