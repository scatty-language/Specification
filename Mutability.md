# Mutability

* MUST treat all local variables and standard method parameters as mutable by default.
* MUST treat the instance receiver (`@`) as immutable by default unless explicitly declared as `ref @`.
* MAY reassign variables or mutate their fields freely when declared via `=` or `:=`.
* MUST isolate mutations to the bound variable without affecting other variables.
* MUST execute a bitwise copy upon assignment of Trivial Types.
* MUST retain a shared reference via ARC upon assignment of Owning Types and automatically execute a Copy-on-Write (CoW) deep copy prior to any mutation if the underlying instance has a strong reference count greater than `1`.

```C#
x := 10;
x = 20;

pointA := Point.New(0, 0);
pointB := pointA;
pointB.X = 5;
// 'pointA.X' remains 0

listA := [+ Int].New();
listB := listA; // ARC retain (strongCount == 2)
listB.Add(5);   // CoW detaches 'listB', 'listA' remains empty
```

## Static Variables

* MAY declare a thread-local static variable within a method using the `static` modifier after the variable's explicit type (`local_id: Type static = Expression;`).
* MAY declare an atomic process-wide global static variable within a method using the `astatic` modifier after the variable's explicit type (`local_id: Type astatic = Expression;`).
* MAY declare a raw unsynchronized process-wide global static variable within a method using the `gstatic` modifier after the variable's explicit type (`local_id: Type gstatic = Expression;`).
* MUST require an explicit type annotation when declaring a `static`, `astatic`, or `gstatic` local variable.
* MUST allocate `static` local variables within Thread-Local Storage (TLS) and evaluate their initialization expression once per thread upon initial execution of the declaration on that thread.
* MUST allocate `astatic` and `gstatic` local variables within the global static data segment shared across all threads and evaluate their initialization expression exactly once upon the initial execution of the declaration in the process.
* MAY read and mutate `static` and `astatic` local variables in safe code without an `unsafe` block.
* MUST compile reads, writes, and compound assignments on scalar primitive `astatic` local variables (up to 64 bits) as hardware atomic operations, and synchronize composite or Owning Type `astatic` local variables via atomic reference retention and Copy-on-Write read-modify-write semantics.
* MUST restrict all reads, mutations, and references to `gstatic` local variables exclusively to `unsafe` blocks or `unsafe` methods.
* MUST NOT pass a `static` or `astatic` local variable directly as a `ref` argument to a method parameter.

```C#
GenerateThreadLocalId() -> Int 
{
    counter: Int static = 0;
    
    counter += 1;
    return counter;
}

RecordGlobalEvent() -> Int 
{
    totalEvents: Int astatic = 0;

    totalEvents += 1;
    return totalEvents;
}

TouchRawState() 
{
    rawCounter: Int gstatic = 0;

    unsafe 
    {
        rawCounter += 1;
    }
}
```

## Method Parameters

* MUST treat parameters passed without the `ref` keyword as logically independent of the caller's variable.
* MUST pass parameters whose memory footprint exceeds the native pointer size via a hidden pointer under the hood without copying memory if the parameter is only read.
* MAY mutate standard parameters internally within the method scope.
* MUST materialize an isolated local copy (or trigger Copy-on-Write detachment for Owning Types) at the site of mutation when a standard parameter is modified inside the method.
* MUST NOT reflect internal parameter mutations on the caller's original variable.

```C#
Process(val: Int, counter: Int) 
{
    val = 5;
    counter = 5;
}

myVal := 10;
myCounter := 0;

Process(myVal, myCounter);
// 'myVal' is 10
// 'myCounter' is 0
```

## Method Receivers

* MUST treat an unqualified `@` receiver as an immutable view of the caller's instance.
* MUST NOT mutate fields on an unqualified `@` receiver or invoke `ref @` methods on it.
* MUST borrow the original instance from the caller for in-place receivers (`ref @`).
* MUST apply mutations on an in-place receiver (`ref @`) directly to the caller's original instance.

```C#
Data 
{
    X: Int;
    
    // '@' is immutable; returns a modified copy via 'with'
    WithX(@, x: Int) -> @ => @ with { X: x };
    
    // 'ref @' mutates the caller's instance in-place
    SetX(ref @, x: Int) 
    {
        @.X = x;
    }
}
```

## Constants

* MUST declare compile-time constants using the double-colon binding operator (`::`).
* MUST evaluate the constant's initialization expression at compile time.
* MUST NOT reassign a constant, mutate its fields, or pass it to a `ref` parameter.
* MAY omit the explicit type annotation (`: Type`) if the compiler can infer the type from the initialization expression.
* MUST govern constant access rights via standard identifier visibility rules (`PublicId`, `InternalId`, or `PrivateId` for file-level and type-level constants, and `LocalId` for method-local constants).

```C#
Pi: Float :: 3.14159f;

MaxUsers :: 100;

DoubleMax :: MaxUsers * 2;
```
