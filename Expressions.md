# Expressions

## Implicit Member Resolution

* MAY omit the explicit type qualifier and use leading dot syntax (`.Member`) when the expected type can be deduced unambiguously from context.
* MUST rewrite an expression of the form `.Member` as `ExpectedType::Member` during compilation.
* MUST resolve a `.Member` identifier to a static method, a static property, or a variant constructor of the expected type.
* MUST NOT resolve instance methods or instance fields via the leading dot syntax.
* MUST recognize the expected type in the following contexts:
  * Variable declarations specifying an explicit type annotation (`variable: Type = .Member`).
  * Assignments to previously typed variables or fields (`variable = .Member`).
  * Invocations where the target parameter type is explicitly declared.
  * Explicit `return` expressions in methods declaring a concrete return type.
  * Field initializers within structural `Data` instantiations where the target field type is known.
* MUST NOT use the leading dot syntax in contexts where the target type is inferred via declaration assignment (`:=`).
* MUST reject any `.Member` expression that cannot resolve to a unique member on the expected type with a compile-time error.

```C#
Vector 
{
    X: Int;
    Y: Int;
    
    Zero: Vector => Vector { X: 0, Y: 0 };
    New(x: Int, y: Int) -> @ => Vector { X: x, Y: y };
}

Option<T> 
{
    ::None;
    ::Some(T);
}

Process(v: Vector, opt: Option<Int>) { }

// Valid: Explicit type annotations
vec: Vector = .Zero;

// Valid: Explicit parameter types
Process(.New(1, 1), .Some(10));

// Valid: Known field types
wrapper := SomeData 
{
    Pos: .Zero 
};

// Valid: Known return type
GetDefault() -> Option<Int> 
{
    return .None;
}

// Invalid: Inferred context lacks target type
ambiguous := .Zero; 

// Invalid: WithX is an instance method, not a static member
wrong: Vector = .WithX(5); 
```
