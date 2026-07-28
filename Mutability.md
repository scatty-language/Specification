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
// 'pointA.X' is still 0
```

## Method Parameters

Function and method parameters act as fully mutable local variables.

* **Call-by-Value Mutability**: When a variable is passed to a method without the `ref` keyword, the method receives a logically independent copy. The method MAY mutate this parameter internally.
* **Caller Independence**: Mutations applied to a standard parameter affect ONLY the local scope of the method. The original variable passed by the caller remains absolutely unchanged.

```C#
Process(val: Int, counter: Int) 
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

* **Local Receiver (`@`)**: The method receives a local, logically independent copy of the instance. The method MAY mutate fields on `@`, but these mutations will disappear when the method returns, unless returned via a fluent interface (`-> @`).
* **In-Place Receiver (`ref @`)**: The method borrows the original instance from the caller. Any mutations applied to `@` modify the original instance directly.

```C#
Data 
{
    X: Int;
    
    // Mutates a local copy
    WithX(@, x: Int) -> @
    {
        @.X = x;
    }
    
    // Mutates the original caller's instance
    SetX(ref @, x: Int) 
    {
        @.X = x;
        
        @ = Data.New(); // ERROR: Reassigning the receiver itself is forbidden
    }
}
```

## Constants

If absolute immutability and compile-time evaluation are required, the Double-colon `::` binding operator MUST be used instead of standard assignment.

```EBNF
ConstDecl ::= Identifier [":" Type] "::" Expression ";"
```

* **Compile-Time Binding**: The RHS `Expression` MUST be evaluable at compile time.
* **Strict Immutability**: A constant binding is permanent. It CANNOT be reassigned, its fields MUST NOT be mutated, and it CANNOT be passed to a `ref` parameter.
* **Type Inference**: The type annotation `: Type` MAY be omitted if the compiler can infer it from the expression.

```C#
// Explicitly typed constant
Pi: Float :: 3.14159f;

// Implicitly typed constant
MaxUsers :: 100;

DoubleMax :: MaxUsers * 2;

// ERROR: Reassignment is forbidden
Pi = 3.14f;

// ERROR: Function call is not a compile-time expression
RandomValue :: Math.Random();
```
