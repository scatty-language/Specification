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
* MUST explicitly type parameters as the interface (`other: ~Interface` or `ref other: ~Interface`) to interact polymorphically with any type implementing the interface.

```scatty
~Equatable 
{
    Equals(@, other: @) -> Bool;

    IsSame(@, other: ~Equatable) -> Bool;
}

~Copyable
{
    Copy(@) -> @;
}
```

## Object Safety and Dispatch Restrictions

* MAY utilize interfaces via static dispatch (generics) or dynamic dispatch (polymorphic parameters and `catch` bindings).
* MUST permit static dispatch calls for ANY method defined in the Interface.
* MUST erase the concrete type and memory size at compile time for dynamic dispatch.
* MUST restrict bare `~Interface` and guarded `~Interface^Guard` types to method parameters (`item: ~Interface` or `ref item: ~Interface`), anonymous function parameters, and `catch` bindings (`catch e: ~Exception`).
* MUST NOT declare a bare `~Interface` or `~Interface^Guard` type as a structural or variant payload field, tuple element, local variable (outside `catch` arms), `static`/`astatic`/`gstatic` storage, method return type, or collection element type.
* MUST treat a non-`ref` interface parameter (`item: ~Interface`) as a read-only polymorphic reference:
  * MAY read interface properties and invoke non-mutating (`@`) interface methods on `item`.
  * MUST NOT invoke mutating (`ref @`) interface methods on `item`, reassign `item`, or pass `item` as a `ref` argument.
* MUST treat a `ref` interface parameter (`ref item: ~Interface`) as a mutable polymorphic reference that MAY invoke both non-mutating (`@`) and mutating (`ref @`) interface methods in-place on the caller's instance.
* MUST permit dynamic dispatch calls ONLY for instance methods that accept `@` or `ref @` and DO NOT return `@` (and do not accept other parameters of type `@`).
* MUST NOT permit dynamic dispatch calls for static methods or methods that reference the concrete self type (`@`) in parameter or return positions.

```scatty
Duplicate<T: ~Copyable>(item: T) 
{
    newItem := item.Copy();
}

InspectDynamic(item: ~Equatable) { }

MutateDynamic(ref item: ~Copyable) { }
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
* MUST NOT invoke or read an unverified weak contract member directly (`item.Method()` or `item.Property`) on an unguarded polymorphic interface parameter (`item: ~Interface` or `ref item: ~Interface`), a `catch` binding (`catch e: ~Interface`), or an unconstrained generic parameter (`T: ~Interface`).
* MAY verify the presence of a weak contract member at runtime using the `is` operator (`must item is .Method` or `must item is .Property`), which checks that the corresponding `vtable` slot is non-null and implicitly refines `item` within the conditional branch.
* MAY declare type guards over an interface (`^Guard = ~Interface must @ is .Member` or `^Guard = @ must @ is .Member`) at the file's top level.
* MAY invoke a weak contract member conditionally using the `?` suffix (`item.Method?(args)` or `item.Property?`):
  * If the member returns a value of type `T`, the expression MUST evaluate to `T?` (`Option::Some` if implemented, or `Option::None` if unimplemented).
  * If the member returns void, the expression MUST execute the method if implemented or perform a no-op if unimplemented.
* MAY invoke a weak contract member forcibly using the `!` suffix (`item.Method!(args)` or `item.Property!`), which MUST immediately trigger runtime Abort or Fatal Panic mechanics if the concrete type does not implement the member.

```scatty
~Stream 
{
    Read(ref @, buffer: [ref Byte..]) throws -> Int;
    Write?(ref @, buffer: [Byte..]) throws;
    
    Seek?(ref @, offset: Long) throws;

    Length?: Long;
}

^Writable = ~Stream must @ is .Write;
^SeekableStream = ~Stream must @ is .Seek and @ is .Length;

// Concrete type implementing only strong contracts and 'Length?'
ReadOnlyMemoryStream : ~Stream 
{
    _data: [Byte];
    Length: Long => @._data.Length as Long;

    Read(ref @, buffer: [ref Byte..]) throws -> Int => 0;
}

ProcessStreams(
    ro: ReadOnlyMemoryStream,
    infoStream: ~Stream,
    ref dynStream: ~Stream,
    ref writable: ~Stream^Writable
) throws 
{
    // 1. Concrete type: statically known at compile time
    len := ro.Length; // Valid: 'Length' is implemented on 'ReadOnlyMemoryStream'
    ro.Write(buffer); // ERROR: 'Write' is not implemented on 'ReadOnlyMemoryStream'

    // 2. Guarded mutable interface parameter: guaranteed to implement 'Write'
    writable.Write(buffer);

    // 3. Unguarded dynamic interface parameter: verified via 'is'
    may dynStream is .Write 
    {
        dynStream.Write(buffer);
    }

    may dynStream is ~Stream^SeekableStream 
    {
        dynStream.Seek(0);
    }

    // 4. Direct optional '?' and forced '!' calls on dynamic interface parameters
    maybeLen: Long? = infoStream.Length?; // Read-only parameter; evaluates to 'Option::None' if unimplemented
    
    dynStream.Write?(buffer);             // No-op if 'Write' is not implemented
    dynStream.Write!(buffer);             // Aborts/Panics if 'Write' is not implemented
}
```
