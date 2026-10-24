# Mutability

* MUST treat all variables, parameters, and method receivers as mutable by default.
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
* MAY declare a process-wide global static variable within a method using the `globstatic` modifier after the variable's explicit type (`local_id: Type globstatic = Expression;`).
* MUST require an explicit type annotation when declaring a `static` or `globstatic` local variable.
* MUST allocate `static` local variables within Thread-Local Storage (TLS) and evaluate their initialization expression once per thread upon initial execution of the declaration on that thread.
* MUST allocate `globstatic` local variables within the global static data segment shared across all threads and evaluate their initialization expression exactly once upon the initial execution of the declaration in the process.
* MAY read and mutate both `static` and `globstatic` local variables without an `unsafe` block.

```C#
GenerateThreadLocalId() -> Int 
{
    counter: Int static = 0;
    
    counter += 1;
    return counter;
}

RecordGlobalEvent() -> Int 
{
    totalEvents: Int globstatic = 0;

    totalEvents += 1;
    return totalEvents;
}

Console.WriteLine(GenerateThreadLocalId()); // Output: 1
Console.WriteLine(GenerateThreadLocalId()); // Output: 2
```

## Method Parameters

* MUST treat parameters passed without the `ref` keyword as logically independent, mutable local copies (Call-by-Value).
* MAY mutate standard parameters internally within the method scope.
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

* MUST instantiate a logically independent copy of the instance for local receivers (`@`).
* MAY mutate fields on a local receiver (`@`).
* MUST NOT propagate local receiver mutations to the caller unless explicitly returned (`-> @`).
* MUST borrow the original instance from the caller for in-place receivers (`ref @`).
* MUST apply mutations on an in-place receiver (`ref @`) directly to the caller's original instance.

```C#
Data 
{
    X: Int;
    
    WithX(@, x: Int) -> @
    {
        @.X = x;
    }
    
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

```C#
Pi: Float :: 3.14159f;

MaxUsers :: 100;

DoubleMax :: MaxUsers * 2;
```
