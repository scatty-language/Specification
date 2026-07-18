# Mutability

In Coggy all variables, parameters, and method receivers are mutable by default.

```EBNF
VarDecl ::= LocalId [":" Type] ("=" | ":=") Expression ";"
```

* **Default Mutability**: Any variable declared via `:=` or `=` is fully mutable. You MAY reassign the variable to a new value or mutate its fields freely.
* **Logical Isolation**: Mutating a variable NEVER affects any other variable, even if one was assigned from the other. The compiler implicitly handles Copy-on-Write (CoW) for heap-backed data.

```C#
x := 10;
x = 20;

pointA := Point.New(0, 0);
pointB := pointA;

pointB.X = 5; // Mutates 'pointB'
// 'pointA.X' is still strictly 0
```

## Method Parameters

Function and method parameters act as fully mutable local variables.

* **Call-by-Value Mutability**: When a variable is passed to a method without the `ref` keyword, the method receives a logically independent copy. The method MAY mutate this parameter internally.
* **Caller Independence**: Mutations applied to a standard parameter affect ONLY the local scope of the method. The original variable passed by the caller remains absolutely unchanged.

```C#
Process(val: int, counter: int) 
{
    val = 5;     // Mutates the local copy
    counter = 5; // Mutates the local copy
}

myVal := 10;
myCounter := 0;

Process(myVal, myCounter);
// 'myVal' is still 10, 'myCounter' is still 0
```

## Method Receivers

Methods operating on an instance dictate whether they mutate a local copy or the caller's original instance.

* **Local Receiver (`@`)**: The method receives a local, logically independent copy of the instance. The method MAY mutate fields on `@`, but these mutations will strictly disappear when the method returns, unless returned via a fluent interface (`-> @`).
* **In-Place Receiver (`ref @`)**: The method borrows the original instance from the caller. Any mutations applied to `@` strictly modify the original instance directly.

```C#
Data 
{
    X: int;
    
    // Mutates a local copy
    WithX(@, x: int) -> @
    {
        @.X = x;
    }
    
    // Mutates the original caller's instance
    SetX(ref @, x: int) 
    {
        @.X = x;
        
        @ = Data.New(); // ERROR: Reassigning the receiver itself is forbidden
    }
}
```

## Constants

If absolute immutability is required, the `#` sigil MUST be used.

```EBNF
ConstDecl ::= "#" Identifier [":" Type] "=" Expression ";"
```

* **Compile-Time Evaluation**: A constant variable MUST be evaluable at compile time.
* **Strict Immutability**: A constant binding can NEVER be reassigned, and its fields MUST NOT be mutated. It CANNOT be passed to a `ref` parameter.

```C#
#Pi: float = 3.14159f;

#Pi = 3.14f; // ERROR: Constants are immutable
```
