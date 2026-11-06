# Expressions

## Implicit Member Resolution

* MAY omit the explicit type qualifier and use the leading-dot syntax (`.Member`) when the expected type (`ExpectedType`) is statically known from context.
* MUST resolve `.Member` according to the declaration kind of `Member` within `ExpectedType`:
  * If `Member` is a variant declared on `ExpectedType`, the compiler MUST rewrite `.Member` (including any variant payload arguments and structural initialization body) as the variant subtype instantiation `ExpectedType::Member`.
  * If `Member` is a static method, static property, static field, or type-scoped constant declared on `ExpectedType`, the compiler MUST rewrite `.Member` as the static member access `ExpectedType.Member`.
* MUST NOT resolve instance methods, instance properties, or instance fields via the leading-dot syntax.
* MUST recognize the expected type in the following contexts:
  * Variable and constant declarations with an explicit type annotation (`variable: Type = .Member;`).
  * Assignments to previously typed variables, fields, or indexed elements (`variable = .Member;`).
  * Method, variant payload, and indexer arguments where the target parameter or key type is explicitly known (`Process(.New(1, 1))`, `weights[.Blue]`).
  * Return expressions in methods, anonymous functions, or `spawn` blocks with a known return type (`return .None;`).
  * Structural field initializers and copy-update (`with`) assignments where the target field type is known (`Pos: .Zero`).
  * Collection and associated array initializer entries where the element, key, or value type is known (`{ .Red = 10 }`).
  * Right-hand operands of binary equality (`==`, `!=`) and coalescing (`??`) operators where the left-hand operand's type is statically known (`status == .Active`, `opt ?? .None`).
  * Arms of a `match` expression where the expected result type of the `match` construct is statically known.
* MUST NOT use the leading-dot syntax in contexts where the target type is inferred (`:=`) without an explicit type expectation.
* MUST reject any `.Member` expression that cannot resolve unambiguously to a single variant or static member on the expected type with a compile-time error.

```scatty
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

// Valid: Resolves '.Zero' to static property 'Vector.Zero'
vec: Vector = .Zero;

// Valid: Resolves '.New(1, 1)' to 'Vector.New(1, 1)' and '.Some(10)' to 'Option<Int>::Some(10)'
Process(.New(1, 1), .Some(10));

// Valid: Known field type inside structural initialization
wrapper := SomeData 
{
    Pos: .Zero 
};

// Valid: Known return type resolves '.None' to 'Option<Int>::None'
GetDefault() -> Option<Int> 
{
    return .None;
}

// Error: Inferred context lacks an expected target type
ambiguous := .Zero; 

// Error: 'WithX' is an instance method, not a static member or variant
wrong: Vector = .WithX(5); 
```
