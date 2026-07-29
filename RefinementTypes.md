# Refinement Types

Refinement types allow developers to attach logical invariants to existing types.

Refinements MUST evaluate to a pure, side-effect-free boolean expression in their `where` clause.

Refinements MUST be declared at the top level of a file.

```EBNF
RefinementDecl ::= "^" Identifier ["<" TypeParams ">"] "=" TargetType [":" ParentList] "where" Expression ";"
TargetType     ::= "@" | Type
TypeParams     ::= TypeParameter ("," TypeParameter)*
```

* **Target Type (TargetType)**: The declaration MUST explicitly specify the base type it refines after the `=` sign.
* **Universal Base Type**: Using `@` as the `TargetType` creates a universal refinement that can be applied to ANY type satisfying the operations in the where expression.
* **Instance Receiver**: Inside the where expression, `@` represents the evaluated instance itself.
* **Explicit Generics**: Developers MAY use explicit generic parameters and assign `T` as the `TargetType` to enforce behavioral contracts.

```C#
// Universal refinement: applicable to any type supporting the '>' operator
^Positive = @ where @ > 0;

// Universal refinement with an explicit interface constraint
^PositiveNumber<T: ~INumber> = T where @ > 0;

// Target-specific refinement: bound to 'String'
^Email = String where @.Contains("@");
```

## Refinement Inheritance and Composition

A refinement type MAY inherit other refinements using the `:` syntax.

* **Implicit Base Type Passing**: When inheriting, the base type of the current refinement is implicitly forwarded to the parent refinements. If explicit generic arguments are needed for the parent, they can be provided via `<@>` or specific types.

```C#
^Positive = @ where @ > 0;

// Inherits '^Positive'
^Big = @ : ^Positive where @ > 1000;

// Complex composition using multiple parents and explicit forwarding
^Complex = @ : ^Positive, ^AnotherComplex<@> where @ % 2 == 0;
```

## Applying Refinements

At the call site, parameter list, or variable declaration, a refinement is applied to a base type using the postfix `^` operator.

```EBNF
RefinedType ::= BaseType "^" Identifier ["<" TypeArgs ">"]
```

* **Base Type Binding**: The `BaseType` defines the memory layout. The `^Refinement` defines the compile-time or runtime invariant.
* **Implicit First Argument**: When writing `BaseType^GenericRefinement`, the `BaseType` is implicitly passed as the primary target type to the refinement.

```C#
validAge: Int^Positive = 25;
email: String^Email = "test@domain.com";

ProcessAge(age: Int^Positive) 
{
    // 'age' is guaranteed to be an Int > 0
}

ProcessBigNumber(num: Float^Big) 
{
    // 'num' is a Float, > 0 and > 1000
}
```

## Validation and Casting

Because refinements do not alter the memory layout, casting between a base type and a refined type requires invariant validation.

* **Implicit Downgrade**: A refined type (`Int^Positive`) CAN be implicitly passed where its base type (`Int`) is expected.
* **Implicit Upgrade Prohibition**: A base type MUST NOT be implicitly passed where a refined type is expected. The invariant MUST be explicitly validated.

### Operators

1. **`is` (Boolean Check)**: Evaluates the invariant. Inside conditionals, it implicitly refines the type.
2. **`as` (Throwing Cast)**: Validates the invariant at runtime. Throws `CastException` if `false`.
3. **`as!` (Unchecked Cast)**: Forces the compiler to treat the base type as refined without runtime validation.
4. **`as?` (Option Cast)**: Validates and returns `Option<BaseType^Refinement>`.

```C#
rawValue: Int = -10;

// Boolean check
if rawValue is Int^Positive 
{
    ProcessAge(rawValue); // Implicitly refined in this block
}

// Throwing cast
valid := rawValue as Int^Positive; // Throws CastException

// Option cast
match rawValue as? Int^Positive 
{
    .Some(val) => ProcessAge(val),
    .None => Console.WriteLine("Invalid!")
}
```

## Mutability and Operations

* **Operator Downgrade**: Standard operators (`+`, `-`) are methods of the base type. Performing an operation on a refined type implicitly downgrades it. The result is the base type.
* **Reassignment Restriction**: You CANNOT implicitly assign the result of an operation back to a mutable refined variable. It MUST be explicitly re-validated.

```C#
MutateAge(ref age: Int^Positive) 
{
    value := age - 1;       // Downgrades to 'Int'
    age = value;            // ERROR: Cannot implicitly assign 'Int' to 'ref Int^Positive'
    
    // Explicit re-validation required:
    age = (age - 1) as Int^Positive; 
}
```
