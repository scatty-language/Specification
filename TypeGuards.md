# Type Guards

* MAY declare a type guard at the file's top level, within a `~Interface` declaration, or within a `+Type` extension block using the prefix `^` operator (`^Identifier = TargetType where Expression;`).
* MUST evaluate the `where` clause expression to a pure, side-effect-free `Bool`.
* MUST explicitly specify the base type being guarded immediately following the `=` token.
* MAY specify a `Data` type, an `~Interface` type, a generic type parameter, or `@` as the `TargetType` of a type guard declaration.
* MAY specify `@` as the `TargetType` to create a universal type guard applicable to any type supporting the operations in the `where` expression.
* MUST evaluate `@` within the `where` expression as the immutable instance receiver.
* MAY declare explicit generic parameters and assign a type parameter as the `TargetType`.

```C#
^Positive = @ where @ > 0;

^PositiveNumber<T: ~INumber> = T where @ > 0;

^Email = String where @.Contains("@");

^Writable = ~Stream where @ is .Write;
```

## Type Guard Inheritance and Composition

* MAY inherit from other type guards using the `:` operator.
* MUST implicitly forward the current type guard's base type to inherited parent type guards.
* MAY provide explicit generic arguments to parent type guards using `<@>` or concrete type identifiers.
* MUST evaluate all inherited parent `where` invariants in declaration order prior to evaluating the child type guard's own `where` expression.

```C#
^Positive = @ where @ > 0;

^Big = @ : ^Positive where @ > 1000;

^Complex = @ : ^Positive, ^AnotherComplex<@> where @ % 2 == 0;
```

## Applying Type Guards

* MUST apply a type guard to a base type using the postfix `^` operator (`BaseType^Guard`).
* MUST define the physical memory layout solely via the `BaseType`.
* MUST evaluate the `^Guard` as a compile-time or runtime invariant.
* MUST implicitly pass the `BaseType` as the primary target type argument to the applied type guard.
* MAY apply a type guard directly to an instance method receiver (`@^Guard` or `ref @^Guard`).
* MUST require the caller's target expression to be statically typed or narrowed to `BaseType^Guard` when invoking a method declared with an `@^Guard` or `ref @^Guard` receiver.

```C#
validAge: Int^Positive = 25 as ^;
email: String^Email = "test@domain.com" as ^;

ProcessAge(age: Int^Positive) { }

ProcessBigNumber(num: Float^Big) { }

Account 
{
    Balance: Int;
}

^Solvent = Account where @.Balance > 0;

+Account 
{
    Withdraw(ref @^Solvent, amount: Int) 
    {
        @.Balance -=^ amount;
    }
}

rawAcc := Account { Balance: 100 };
rawAcc.Withdraw(50); // ERROR: 'rawAcc' is 'Account', not 'Account^Solvent'

if rawAcc is Account^Solvent 
{
    rawAcc.Withdraw(50); // Valid: 'rawAcc' is narrowed to 'Account^Solvent'
}
```

## Validation and Casting

* MAY implicitly downgrade a guarded type (`BaseType^Guard`) to its unguarded `BaseType` when passed by value.
* MUST NOT implicitly cast an unguarded `BaseType` to a guarded type (`BaseType^Guard`).
* MUST explicitly validate the invariant when casting from a base type to a guarded type, unless using `unas` or a guarded assignment operator (`=^`).

### Operators

* MUST evaluate the type guard's `where` invariant at compile time whenever the operand of `as`, `as?`, `as!`, or `=^` is a compile-time constant expression:
  * If the compile-time evaluation yields `true`, the compiler MUST elide all runtime validation branches.
  * If the compile-time evaluation yields `false`, the compiler MUST reject the expression with a compile-time error (or evaluate `as?` directly to `Option::None`).
* MUST evaluate the invariant and implicitly narrow the type within conditional blocks when using the `is` operator.
* MUST validate the invariant at runtime (in both Debug and Release modes, unless proven at compile time) and throw a `CastException` upon failure when using the `as` operator.
* MUST validate the invariant at runtime and yield `Option<BaseType^Guard>` when using the `as?` operator.
* MUST validate the invariant at runtime (in both Debug and Release modes, unless proven at compile time) and immediately trigger runtime Abort or Fatal Panic mechanics upon failure when using the `as!` operator.
* MUST bypass runtime validation and force compiler acceptance when using the `unas` operator.
* MAY substitute the explicit target type with the contextual guard symbol (`^`) following `as`, `as?`, `as!`, or `unas` when the expected guarded type is statically known from context (explicitly typed declarations, assignments, method parameter types, `return` statements, or structural field initializers).
* MUST reject the contextual guard cast (`as ^`, `as? ^`, `as! ^`, `unas ^`) at compile time if the target context does not expect a guarded type or if the expected type is inferred (`:=`).
* MAY use a type guard pattern (`^Guard`) directly within a `match` arm to validate the invariant and narrow the matched subject within the arm's scope.

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

// Runtime contextual guard cast: 'Int^Positive' deduced from 'ProcessAge' parameter
ProcessAge((rawValue + 20) as ^);

match rawValue as? Int^Positive 
{
    .Some(val) => ProcessAge(val),
    .None => Console.WriteLine("Invalid!")
}
```

## Mutability and Invariant Protection

* MUST implicitly downgrade the evaluation result of base type operators (e.g., `age - 1`) to the unguarded base type.
* MUST NOT assign an unguarded evaluation result directly to a mutable guarded variable using standard assignment operators (`=`, `+=`, `-=`, `*=`, `/=`, `%=`).
* MAY use the guarded assignment operators (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`) to evaluate the right-hand expression and assign the result to a guarded variable or field.
* MUST restrict guarded assignment operators (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`) to targets governed by a type guard and right-hand expressions evaluating to the exact underlying base type.
* MUST NOT invoke custom `~Castable<T>` transformations during a guarded assignment (`=^`).
* MUST require an explicit guarded type annotation (`: Type^Guard`) when using `=^` in a variable declaration.

### Field Mutations and In-Place Operations on Guarded Variables

* MUST NOT mutate an internal field of a guarded `Data` variable (`instance: Data^Guard`) using standard assignment operators (`instance.Field = val`).
* MUST require guarded assignment operators (`instance.Field =^ val`, `instance.Field +=^ val`) when mutating any field of a variable governed by a type guard, which re-evaluates the enclosing `Data^Guard` invariant after updating the field.
* MUST require guarded assignment operators (`@.Field =^ val`, `@.Field +=^ val`) when mutating any field of `@` within a method declaring a `ref @^Guard` receiver, re-evaluating the `^Guard` invariant according to the active build-mode validation rules.
* MUST automatically re-evaluate the `where` invariant of a guarded variable (`instance: Type^Guard`) immediately after:
  * Invoking any mutating (`ref @`) method on `instance` (e.g., `list.Clear()` on `list: [+ Int]^NonEmpty`), or
  * Passing `instance` (or any of its fields) as a `ref` argument to a parameter typed as the unguarded base type.
* MUST enforce the following build-mode validation semantics for all statement-level guarded assignments (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`), guarded field mutations, and post-`ref` invariant checks when the invariant cannot be statically proven at compile time:
  * In Debug mode, the compiler MUST evaluate the governing `where` invariant at runtime immediately after the mutation and trigger a Fatal Panic if the invariant evaluates to `false`.
  * In Release mode, the compiler MUST elide the runtime invariant check by default, executing the mutation with zero branching overhead.
  * Implementations MUST provide a compiler configuration flag that retains runtime invariant checks for all guarded mutations in Release mode, triggering an immediate process Abort upon invariant violation.
* MUST always evaluate the runtime invariant check in both Debug and Release modes when `=^` is used inside a conditional assignment (`if target =^ expr`).

```C#
User 
{
    Name: String;
    Age: Int;
}

^Adult = User where @.Age >= 18;
^NonEmpty = [+ Int] where @.Count > 0;

ResetUser(ref u: User) 
{
    u.Age = 5;
}

MutateExamples(ref age: Int^Positive, ref adult: User^Adult, ref list: [+ Int]^NonEmpty) 
{
    // 1. Guarded scalar mutation: checked in Debug, zero-cost in default Release
    age -=^ 10;

    // 2. Guarded struct field mutation:
    // 'adult.Age = 10;' is a COMPILE-TIME ERROR; must use '=^'
    // Re-verifies 'adult' against '^Adult' ('@.Age >= 18') in Debug / strict mode
    adult.Age =^ 21;

    // 3. Mutating 'ref @' method on a guarded collection:
    // Automatically asserts '^NonEmpty' ('@.Count > 0') after 'RemoveAt' in Debug / strict mode
    list.RemoveAt(0);

    // 4. Passing a guarded variable by 'ref' to an unguarded 'ref User' parameter:
    // Automatically asserts '^Adult' on 'adult' upon return in Debug / strict mode
    ResetUser(ref adult); // Triggers Fatal Panic in Debug mode because Age became 5
}
```
