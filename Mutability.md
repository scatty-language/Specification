# Mutability

In Coggy, all variables, parameters, and method receivers are strictly **immutable by default**. The `mut` keyword MUST be explicitly provided to allow reassignment or field mutation.

## Variable Declarations

```EBNF
VarDecl ::= ["mut"] Identifier [":" Type "="] | ":=" Expression ";"
```

* **Immutable Binding**: A variable declared without `mut` is permanently bound to its initial value or instance. Any attempt to reassign the variable or mutate its fields is a compilation error.
* **Mutable Binding**: The `mut` keyword permits both reassignment of the variable to a new value and the mutation of its internal fields.
* **Transitive Immutability**: Immutability is deep. If an instance is bound immutably, all its fields are strictly treated as immutable, regardless of how they are defined in the `Data` declaration.

```C#
x: int = 10;
x = 20;      // Error: 'x' is immutable

mut y := 10;
y = 20;      // Valid: 'y' is mutable

point := Point.New(0, 0);
point.X = 5; // Error: 'point' is immutable

mut mutPoint := Point.New(0, 0);
mutPoint.X = 5; // Valid
```

## Function Parameters

* **Call-by-Value Immutability**: Function parameters act as local bindings. If the `mut` keyword is omitted, the parameter MUST NOT be reassigned or mutated within the function body.
* **Caller Independence**: The `mut` modifier on a parameter affects ONLY the local scope of the function. It DOES NOT affect the mutability of the original variable passed by the caller (unless passed as a mutable view pointer).

```C#
Process(val: int, mut counter: int) 
{
    val = 5;     // Error: 'val' is immutable
    counter = 5; // Valid: 'counter' is mutable
}
```

## Method Receivers

Methods operating on an instance strictly dictate whether the instance can be mutated via the receiver parameter `@`.

* **Read-Only Receiver**: If a method is declared with `@`, the instance is borrowed immutably. The method MUST NOT mutate any fields of the instance.
* **Mutable Receiver**: If a method is declared with `mut @`, the instance is borrowed mutably. The method MAY mutate the fields of the instance.
* **No Receiver Reassignment**: Even within a `mut @` method, reassigning the `@` pointer itself (e.g., `@ = other;`) remains STRICTLY FORBIDDEN.

```C#
Data 
{
    X: int;
    
    // Read-only method
    MakeSome(@) 
    {
        @.X = 20; // Error: @ is immutable, field mutation forbidden
    }
    
    // Mutating method
    MakeGoodSome(mut @) 
    {
        @.X = 100; // Valid: field mutation allowed
        @ = Data.New(); // Error: Reassigning the receiver pointer itself is forbidden
    }
}
```

## Immutable Contexts and Mutability Escalation

* **Escalation Prohibition**: An immutable variable or receiver MUST NOT be passed to a function or method that requires a mutable parameter or a mutable receiver.

```C#
MutateData(mut data: Data) { }

immutableData := Data.New();

// Error: Cannot call mutating method on an immutable instance
immutableData.MakeGoodSome(); 

// Error: Cannot pass immutable instance to a method expecting a mutable binding 
MutateData(immutableData); 
```
