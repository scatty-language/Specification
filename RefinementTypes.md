# Refinement Types

Refinement types allow developers to attach strict logical invariants to existing Data types.

Refinements MUST evaluate to a pure, side-effect-free boolean expression in their `where` clause.

```EBNF
RefinementDecl ::= "^" Identifier ["<" TypeParams ">"] "=" TargetType [":" ParentList] "where" Expression ";"
TargetType     ::= "@" | Type
TypeParams     ::= TypeParameter ("," TypeParameter)*
```

* **The Universal Base Type (`@`)**: In the declaration, using `@` as the `TargetType` signifies that the refinement is generic and can be applied to ANY type that satisfies the `where` expression.
* **The Instance Receiver (`@`)**: Inside the `where` expression, `@` represents the evaluated instance itself.
* **Explicit Generics**: Developers MAY use explicit generic constraints (`T: Number`) instead of the universal `@` if strict trait bounds are required.

```C#
// Universal refinement: applicable to any type supporting the '>' operator
^Positive = @ where @ > 0;

// Explicit generic refinement
^PositiveNumber<T: ~Number> = T where @ > 0;

// Constrained to 'string' type
^Email = string where @.Contains("@");
```

## Refinement Inheritance and Composition

A refinement type MAY inherit other refinements using the `:` syntax.

* **Implicit Base Type Passing**: When inheriting, the base type of the current refinement is implicitly forwarded to the parent refinements. If explicit generic arguments are needed for the parent, they can be provided via `<@>` or specific types.

```C#
^Positive = @ where @ > 0;

// Inherits ^Positive
^Big = @ : ^Positive where @ > 1000;

// Complex composition using multiple parents and explicit forwarding
^Complex = @ : ^Positive, ^AnotherComplex<@> where @ % 2 == 0;
```

## Applying Refinements

At the call site, parameter list, or variable declaration, a refinement is applied to a base type using the postfix `^` operator.

```EBNF
RefinedType ::= BaseType "^" Identifier ["<" TypeArgs ">"]
```

* **Base Type Binding**: The `BaseType` strictly defines the memory layout. The `^Refinement` defines the compile-time or runtime invariant.
* **Implicit First Argument**: When writing `BaseType^GenericRefinement`, the `BaseType` is implicitly passed as the primary target type to the refinement.

```C#
validAge: int^Positive = 25;
email: string^Email = "test@domain.com";

ProcessAge(age: int^Positive) 
{
    // 'age' is guaranteed to be an int > 0
}

ProcessBigNumber(num: float^Big) 
{
    // 'num' is a float, > 0 and > 1000
}
```

## Validation and Casting

Because refinements do not alter the memory layout, casting between a base type and a refined type requires invariant validation.

* **Implicit Downgrade**: A refined type (`int^Positive`) CAN be implicitly passed where its base type (`int`) is expected.
* **Implicit Upgrade Prohibition**: A base type MUST NOT be implicitly passed where a refined type is expected. The invariant MUST be explicitly validated.

### Operators

1. **`is` (Boolean Check)**: Evaluates the invariant. Inside conditionals, it implicitly refines the type.
2. **`as` (Throwing Cast)**: Validates the invariant at runtime. Throws `#CastException` if `false`.
3. **`as!` (Unchecked Cast)**: Forces the compiler to treat the base type as refined without runtime validation.
4. **`as?` (Option Cast)**: Validates and returns `Option<BaseType^Refinement>`.

```C#
rawValue: int = -10;

// Boolean check
if rawValue is int^Positive 
{
    ProcessAge(rawValue); // Implicitly refined in this block
}

// Throwing cast
valid := rawValue as int^Positive; // Throws #CastException

// Option cast
match rawValue as? int^Positive 
{
    .Some(val) => ProcessAge(val),
    .None => Console.WriteLine("Invalid!")
}
```

## Mutability and Operations

* **Operator Downgrade**: Standard operators (`+`, `-`) are methods of the base type. Performing an operation on a refined type implicitly downgrades it. The result is strictly the base type.
* **Reassignment Restriction**: You CANNOT implicitly assign the result of an operation back to a mutable refined variable. It MUST be explicitly re-validated.

```C#
MutateAge(ref age: int^Positive) 
{
    value := age - 1;       // Downgrades to 'int'
    age = value;            // ERROR: Cannot implicitly assign 'int' to 'ref int^Positive'
    
    // Explicit re-validation required:
    age = (age - 1) as int^Positive; 
}
```
