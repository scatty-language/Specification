# Closures and Lambdas

Coggy provides concise syntax for anonymous functions (lambdas). Lambdas natively capture variables from their enclosing lexical scope.

The compiler translates every lambda expression into a transient, anonymous `Data` type.

```EBNF
LambdaExpr      ::= LambdaParams "=>" (Expression | Block)
LambdaParams    ::= LocalId | "(" [LambdaParamList] ")"
LambdaParamList ::= LambdaParam ("," LambdaParam)*
LambdaParam     ::= ["ref"] LocalId [":" Type]
```

## Type Inference

* **Implicit Types**: If the lambda is passed into a strongly typed context (e.g., a generic method with known constraints), the compiler MUST infer the parameter and return types automatically.
* **Single Parameter**: If a lambda has exactly one parameter and its type can be inferred, the parentheses `()` MAY be omitted.
* **Body**: The body can be a single `Expression` (implicitly returned) or a `{}` block.

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

## Zero-Cost Desugaring and Interfaces

When a lambda is declared, the compiler implicitly generates an anonymous `Data` type. This type automatically implements standard library `~Callable` interfaces corresponding to the lambda's signature.

```C#
// Standard library generic interfaces for callables
~Callable<TReturn> { Invoke(@) -> TReturn; }
~Callable1<T1, TReturn> { Invoke(@, arg1: T1) -> TReturn; }
// ...
```

When a lambda is passed to a generic method constrained by a `~Callable` interface, the compiler enforces monomorphization. The lambda's `Invoke` method is statically resolved and inlined.

```C#
Extensions 
{
    // Generic method constrained by a callable interface
    Filter<T, TFunc: ~Callable1<T, Bool>>(list: []T, predicate: TFunc) -> []T 
    {
        result := []T.New();

        for item in list 
        {
            // Direct static dispatch. Statically inlined by the optimizer.
            if predicate.Invoke(item) 
            {
                result.Add(item);
            }
        }

        return result;
    }
}

users := GetUsers();
active := users.Filter(u => u.IsActive); 
```

## Capture and Memory Safety Rules

When a lambda references variables from its enclosing scope, it structurally captures them as fields within its anonymous `Data` type.

1. **Value Capture**: All captured variables are implicitly copied by value into the anonymous lambda structure. Standard Copy-on-Write (CoW) applies to ARC-backed types.
2. **Local Mutation**: A lambda MAY mutate its captured variables, but it mutates its local copy stored inside the anonymous `Data`. The original variable in the enclosing scope remains absolutely unaffected.
3. **Reference Capture Prohibition**: Capturing outer variables by reference (`ref`) is FORBIDDEN. A lambda MUST NOT reference outer variables that were declared or passed as `ref`.

```C#
Process() 
{
    multiplier := 10;
    
    // The compiler generates an anonymous Data containing 'multiplier: Int'
    // 'multiplier' is copied by value.
    scale := (val: Int) => val * multiplier;

    counter := 0;
    increment := () => 
    {
        // Mutates the lambda's internal copy of 'counter'
        counter += 1; 
        Console.WriteLine(counter);
    };

    increment(); // Prints 1
    increment(); // Prints 2
    
    // The original outer variable is completely unchanged
    Console.WriteLine(counter); // Prints 0
}
```

## Dynamic Dispatch (Type Erasure)

If a lambda must be stored in a heterogeneous collection or passed where monomorphization is impossible, it CAN be cast to a dynamic interface wrapper (`Box<~Callable1<T1, TReturn>>`). In this case, standard dynamic dispatch rules apply.

```C#
// Storing lambdas dynamically requires boxing the anonymous Data
handlers: Box<~Callable<Int>>[] = Box[].New();

handlers.Add(Box.New(() => 42));
handlers.Add(Box.New(() => 99));
```
