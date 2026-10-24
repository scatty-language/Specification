# Interfaces

* MUST NOT contain state fields.
* MUST occupy exactly 0 bytes in memory.
* MAY be implemented by a Data type in unlimited quantities.
* MUST NOT be instantiated via structural initialization.

## Method Contracts and Default Implementations

* MAY declare contract methods without a body.
* MUST provide a concrete method with a matching signature in the implementing `Data` type for every contract method.
* MAY omit the `throws` modifier on the implementing method in a `Data` type when satisfying an interface contract method declared with `throws`.
* MUST NOT declare the `throws` modifier on an implementing method if the corresponding interface contract method omits `throws`.
* MUST evaluate direct invocations on a concrete `Data` type according to the concrete method's `throws` modifier.
* MUST evaluate invocations through an interface reference or generic constraint according to the interface contract's `throws` modifier.
* MAY declare default methods with a body to provide shared behavior.
* MAY shadow a default interface method by explicitly declaring a method with an identical signature in the implementing `Data` type.

```C#
~Comparable 
{
    Compare(@, other: @) -> Int;

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
}
```

## Property Contracts and Satisfaction

* MAY declare contract properties to enforce read access.
* MAY declare default computed properties with a body.
* MAY shadow a default computed property by explicitly declaring it in the implementing Data type.
* MUST satisfy a contract property by declaring either a computed property OR a state field with an identical name and type in the implementing Data type.
* MUST implicitly generate a transparent vtable getter to dynamically resolve the field or property when accessed via an interface reference.

```C#
~Vectorable 
{
    X: Int; 
    Y: Int;

    DoubleX: Int => @.X * 2; 

    LengthSquared: Int  
    {
        return (@.X * @.X) + (@.Y * @.Y);
    }
}

Entity : ~Vectorable 
{
    X: Int; 
    Y: Int;
}
```

## Receiver and Self-Resolution

* MUST resolve the receiver within an Interface declaration to the concrete implementing type, not the interface type.
* MUST return an instance of the exact concrete type if a method signature declares a return type of `-> @`.
* MUST explicitly type parameters as the interface to interact polymorphically with any type implementing the interface.

```C#
~Equatable 
{
    Equals(@, other: @) -> Bool;

    IsSame(@, ref other: ~Equatable) -> Bool;
}

~Copyable
{
    Copy(@) -> @;
}
```

## Object Safety and Dispatch Restrictions

* MAY utilize interfaces via static dispatch (generics) or dynamic dispatch (references or containers).
* MUST permit static dispatch calls for ANY method defined in the Interface.
* MUST erase the concrete type and memory size at compile time for dynamic dispatch.
* MUST permit dynamic dispatch calls ONLY for methods that accept `@` or `ref @` and DO NOT return `@`.
* MUST NOT permit dynamic dispatch calls for methods that return the concrete type (`@`).

```C#
Duplicate<T: ~Copyable>(item: T) 
{
    newItem := item.Copy();
}

ProcessDynamic(ref item: ~Copyable) { }
```
