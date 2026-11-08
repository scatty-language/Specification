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

```scatty
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
        may @.Value > other.Value => return 1;
        may @.Value < other.Value => return -1;
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

```scatty
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

```scatty
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

```scatty
Duplicate<T: ~Copyable>(item: T) 
{
    newItem := item.Copy();
}

ProcessDynamic(ref item: ~Copyable) { }
```

## Weak Contracts and Interface Type Guards

* MAY declare a method or property contract within an interface as a weak (optional) contract by appending the `?` suffix to its identifier (`Method?(...)` or `Property?: Type;`).
* MUST NOT declare the `?` weak contract suffix on methods or properties outside of an `~Interface` declaration.
* MAY omit the implementation of any weak contract member in a concrete `Data` type implementing the interface.
* MUST omit the `?` suffix when implementing a weak contract member on a concrete `Data` type.

### Static Resolution on Concrete Types

* MUST resolve access to a weak contract member at compile time when the target expression's concrete `Data` type is statically known:
  * If the concrete `Data` type implements the weak member, the compiler MUST permit direct access (`instance.Member` or `instance.Method()`) with zero runtime existence checks.
  * If the concrete `Data` type omits the weak member, the compiler MUST reject any access (`instance.Member`, `instance.Method()`, `instance.Method?()`, or `instance.Method!()`) with a compile-time error.

### Dynamic Resolution and Type Guards

* MUST represent an unimplemented weak contract member as a null slot (`0x0`) within the interface's virtual method table (`vtable`) during dynamic dispatch.
* MUST NOT invoke or read an unverified weak contract member directly (`box.Method()` or `box.Property`) on an unguarded dynamic interface container (`Box<~Interface>`), an interface reference (`ref item: ~Interface`), or an unconstrained generic parameter (`T: ~Interface`).
* MAY verify the presence of a weak contract member at runtime using the `is` operator (`must item is .Method` or `must item is .Property`), which checks that the corresponding `vtable` slot is non-null and implicitly refines `item` within the conditional branch.
* MAY declare type guards over an interface (`^Guard = ~Interface must @ is .Member` or `^Guard = @ must @ is .Member`) either inside the `~Interface` body or at the file top level.
* MAY invoke a weak contract member conditionally using the `?` suffix (`item.Method?(args)` or `item.Property?`):
  * If the member returns a value of type `T`, the expression MUST evaluate to `T?` (`Option::Some` if implemented, or `Option::None` if unimplemented).
  * If the member returns void, the expression MUST execute the method if implemented or perform a no-op if unimplemented.
* MAY invoke a weak contract member forcibly using the `!` suffix (`item.Method!(args)` or `item.Property!`), which MUST immediately trigger runtime Abort or Fatal Panic mechanics if the concrete type does not implement the member.

```scatty
~Stream 
{
    Read(ref @, buffer: [Byte]) throws -> Int;
    Write?(ref @, buffer: [Byte]) throws;
    Seek?(ref @, offset: Long) throws;
    Length?: Long;

    // Interface type guard declared inside the interface
    ^Writable = @ must @ is .Write;
}

// Interface type guard declared outside at the file top level
^SeekableStream = ~Stream must @ is .Seek and @ is .Length;

// Concrete type implementing only strong contracts and 'Length?'
ReadOnlyMemoryStream : ~Stream 
{
    _data: [Byte];
    Length: Long => @._data.Length as Long;

    Read(ref @, buffer: [Byte]) throws -> Int => 0;
}

ProcessStreams(ro: ReadOnlyMemoryStream, dynStream: Box<~Stream>, writable: Box<~Stream^Writable>) throws 
{
    // 1. Concrete type: statically known at compile time
    len := ro.Length; // Valid: 'Length' is implemented on 'ReadOnlyMemoryStream'
    ro.Write(buf); // ERROR: 'Write' is not implemented on 'ReadOnlyMemoryStream'

    // 2. Guarded interface: guaranteed to implement 'Write'
    writable.Write(buf);

    // 3. Unguarded dynamic interface: verified via 'is'
    may dynStream is .Write 
    {
        dynStream.Write(buf);
    }

    may dynStream is ~Stream^SeekableStream 
    {
        dynStream.Seek(0);
    }

    // 4. Direct optional '?' and forced '!' calls on dynamic interface
    dynStream.Write?(buf);          // No-op if 'Write' is not implemented
    maybeLen: Long? = dynStream.Length?; // Evaluates to 'Option::None' if unimplemented
    dynStream.Write!(buf);          // Aborts/Panics if 'Write' is not implemented
}
```
