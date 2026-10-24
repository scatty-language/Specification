# Closures and Lambdas

* MUST translate every lambda expression into a transient, anonymous `Data` type.
* MUST implicitly implement the corresponding standard library `~Callable` interface on the generated anonymous `Data` type matching the lambda's parameter and return signatures.
* MUST automatically infer parameter and return types when a lambda expression occurs within a strongly typed context.
* MAY omit parentheses around the parameter identifier if a lambda declares exactly one parameter and its type is inferred.
* MUST require parentheses around parameter declarations if the lambda declares zero parameters, two or more parameters, or any explicit type annotations.
* MAY declare a lambda body as either a single `Expression` or a block `{}`.
* MUST implicitly return the evaluated value of a single `Expression` body.
* MUST require an explicit `return` statement to yield a value from a block `{}` body.

```C#
// Single parameter, inferred type, expression body
square := x => x * x;

// Multiple parameters, explicit types, block body
add := (a: Int, b: Int) => 
{
    return a + b;
};

// Parameter passed by reference
increment := (ref val: Int) => val += 1;
```

## Interface Implementation and Dispatch

* MUST statically resolve and inline invocations of a lambda's `Invoke` method when passed to a generic method constrained by a `~Callable` interface.
* MUST monomorphize generic functions parameterized over a `~Callable` interface.
* MAY box a lambda instance into `Box<~Callable<...>>` to permit storage in heterogeneous collections or dynamic dispatch.
* MUST apply dynamic dispatch rules to invocations on boxed `~Callable` instances.

```C#
Filtration 
{
    Filter<T, TFunc: ~Callable<T, Bool>>(list: [+ T], predicate: TFunc) -> [+ T] 
    {
        result := [+ T].New();

        for item in list 
        {
            if predicate.Invoke(item) 
            {
                result.Add(item);
            }
        }

        return result;
    }
}

users := GetUsers();
active := Filtration.Filter(users, u => u.IsActive); 

// Boxing lambdas for dynamic dispatch
handlers := [+ Box<~Callable<Int>>].New();
handlers.Add(Box.New(() => 42));
handlers.Add(Box.New(() => 99));
```

## Variable Capture and Memory Safety

* MUST capture referenced variables from outer lexical scopes as structural fields of the generated anonymous `Data` type.
* MUST capture variables by value.
* MUST NOT capture outer lexical variables by reference (`ref`).
* MUST copy Trivial Types bitwise into the lambda instance upon capture.
* MUST atomically increment the ARC strong reference count (`strongCount`) of Owning Types upon capture.
* MUST NOT invalidate, move, or poison captured variables in the enclosing lexical scope.
* MUST govern mutations of captured Owning Types via Copy-on-Write (CoW) semantics.
* MAY capture an Owning Type via a `weak` reference to prevent ARC reference cycles.

```C#
Process() 
{
    multiplier := 10;
    buffer := [+ Int].New();
    
    // 'multiplier' is bitwise copied
    // 'buffer' strongCount is incremented via ARC
    worker := (val: Int) => 
    {
        buffer.Add(val * multiplier);
    };

    // Both remain accessible in outer scope
    Console.WriteLine(multiplier); 
    Console.WriteLine(buffer.Count); 
}
```
