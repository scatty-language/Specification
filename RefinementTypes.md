# Refinement Types

* MUST declare refinement types at the file's top level.
* SHOULD evaluate the `where` clause expression to a pure, side-effect-free boolean.
* MUST explicitly specify the base type being refined immediately following the `=` token.
* MAY specify `@` as the `TargetType` to create a universal refinement applicable to any type satisfying the `where` expression.
* MUST evaluate `@` within the `where` expression as the instance receiver.
* MAY declare explicit generic parameters and assign a type parameter as the `TargetType`.

```C#
^Positive = @ where @ > 0;

^PositiveNumber<T: ~INumber> = T where @ > 0;

^Email = String where @.Contains("@");
```

## Refinement Inheritance and Composition

* MAY inherit from other refinements using the `:` operator.
* MUST implicitly forward the current refinement's base type to inherited parent refinements.
* MAY provide explicit generic arguments to parent refinements using `<@>` or concrete type identifiers.

```C#
^Positive = @ where @ > 0;

^Big = @ : ^Positive where @ > 1000;

^Complex = @ : ^Positive, ^AnotherComplex<@> where @ % 2 == 0;
```

## Applying Refinements

* MUST apply a refinement to a base type using the postfix `^` operator.
* MUST define the memory layout solely via the `BaseType`.
* MUST evaluate the `^Refinement` as a compile-time or runtime invariant.
* MUST implicitly pass the `BaseType` as the primary target type argument to the applied refinement.

```C#
validAge: Int^Positive = 25;
email: String^Email = "test@domain.com";

ProcessAge(age: Int^Positive) { }

ProcessBigNumber(num: Float^Big) { }
```

## Validation and Casting

* MAY implicitly cast a refined type to its base type.
* MUST NOT implicitly cast a base type to a refined type.
* MUST explicitly validate the invariant when casting from a base type to a refined type, unless using `unas`.

### Operators

* MUST evaluate the refinement's `where` invariant at compile time whenever the operand of `as`, `as?`, `as!`, or `=^` is a compile-time constant expression:
  * If the compile-time evaluation yields `true`, the compiler MUST elide all runtime validation branches.
  * If the compile-time evaluation yields `false`, the compiler MUST reject the expression with a compile-time error (or evaluate `as?` directly to `Option::None`).
* MUST evaluate the invariant and implicitly refine the type within conditional blocks when using the `is` operator.
* MUST validate the invariant at runtime (in both Debug and Release modes, unless proven at compile time) and throw a `CastException` upon failure when using the `as` operator.
* MUST validate the invariant at runtime and yield `Option<BaseType^Refinement>` when using the `as?` operator.
* MUST validate the invariant at runtime (in both Debug and Release modes, unless proven at compile time) and immediately trigger runtime Abort or Fatal Panic mechanics upon failure when using the `as!` operator.
* MUST bypass runtime validation and force compiler acceptance when using the `unas` operator.
* MAY substitute the explicit target type with the contextual refinement symbol (`^`) following `as`, `as?`, `as!`, or `unas` when the expected refined type is statically known from context (explicitly typed declarations, assignments, method parameter types, `return` statements, or structural field initializers).
* MUST reject the contextual refinement cast (`as ^`, `as? ^`, `as! ^`, `unas ^`) at compile time if the target context does not expect a refined type or if the expected type is inferred (`:=`).
* MAY use a refinement pattern (`^Refinement`) directly within a `match` arm to validate the invariant and refine the matched subject within the arm's scope.

```C#
rawValue: Int = -10;

if rawValue is Int^Positive 
{
    ProcessAge(rawValue);
}

// Throws CastException if rawValue <= 0
valid := rawValue as Int^Positive;

// Aborts/Panics immediately if rawValue <= 0
forced := rawValue as! Int^Positive;

// Unchecked cast: bypasses runtime validation
uncheckedVal := rawValue unas Int^Positive;

// Compile-time evaluated: '11 > 0' is proven at compile time -> ZERO runtime branches!
ProcessAge(11 as ^);

// Compile-time ERROR: '-5 > 0' evaluates to false at compile time
// ProcessAge(-5 as ^);

// Runtime contextual refinement cast: 'Int^Positive' deduced from 'ProcessAge' parameter
ProcessAge((rawValue + 20) as ^);

match rawValue as? Int^Positive 
{
    .Some(val) => ProcessAge(val),
    .None => Console.WriteLine("Invalid!")
}
```

## Mutability and Operations

* MUST implicitly downgrade the evaluation result of base type operators to the unrefined base type.
* MUST NOT implicitly assign an unrefined evaluation result to a mutable refined variable using standard assignment operators (`=`, `+=`, `-=`, `*=`, `/=`, `%=`).
* MAY use the refined assignment operators (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`) to evaluate the right-hand expression and assign the result to a refined target variable or field.
* MUST restrict refined assignment operators (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`) to targets declared with a refined type and right-hand expressions evaluating to the exact underlying base type.
* MUST NOT invoke custom `~Castable<T>` transformations during a refined assignment (`=^`).
* MUST require an explicit refined type annotation (`: Type^Refinement`) when using `=^` in a variable declaration.
* MUST enforce the following build-mode validation semantics for statement-level refined assignments (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`) when the invariant cannot be statically proven at compile time:
  * In Debug mode, the compiler MUST evaluate the target's `where` invariant at runtime and immediately trigger a Fatal Panic if the invariant evaluates to `false`.
  * In Release mode, the compiler MUST elide the runtime invariant check by default, executing the assignment with zero branching overhead.
  * Implementations MUST provide a compiler configuration flag (e.g., strict contract enforcement) that retains runtime invariant checks for all refined assignment operators in Release mode, triggering an immediate process Abort upon invariant violation.
* MUST always evaluate the runtime invariant check in both Debug and Release modes when `=^` is used inside a conditional assignment (`if target =^ expr`), as control flow depends on the validation result.

```C#
MutateAge(ref age: Int^Positive) 
{
    // Downgrades to base type 'Int'
    value: Int = age - 1;
    
    // Explicit 'as' cast: validated at runtime in both Debug and Release modes
    age = (age - 1) as ^;

    // Refined assignment:
    // - Validated at runtime in Debug mode (triggers Fatal Panic if <= 0)
    // - Zero-cost (branchless) in Release mode by default, unless strict contract flag is enabled
    age =^ age - 1;
    age -=^ 10;
}

InitAge(raw: Int) 
{
    // Constant expression: validated at compile time, zero runtime branches
    constAge: Int^Positive =^ 10;

    // Dynamic expression: checked in Debug, zero-cost in default Release
    validAge: Int^Positive =^ raw + 10;
}
```
