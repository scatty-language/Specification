# Type Guards

* MUST declare a type guard at the file's top level using the prefix `^` operator (`^Identifier = TargetType must Expression;`).
* MUST NOT declare a type guard within a `Data` declaration, an `~Interface` declaration, or a `with` extension block.
* MUST evaluate the `must` clause expression to a pure, side-effect-free `Bool`.
* MUST explicitly specify the base type being guarded immediately following the `=` token.
* MAY specify a `Data` type, an `~Interface` type, a generic type parameter, or `@` as the `TargetType` of a type guard declaration.
* MAY specify `@` as the `TargetType` to create a universal type guard applicable to any type supporting the operations in the `must` expression.
* MUST evaluate `@` within the `must` expression as the immutable instance receiver.
* MAY declare explicit generic parameters and assign a type parameter as the `TargetType`.
* MAY omit the base type and use the contextual guard pattern (`is ^Guard` or `is ^Guard<TypeArgs>`) when the type guard is applicable to the left-hand expression's static type.

```scatty
^Positive = @ must @ > 0;

^PositiveNumber<T: ~INumber> = T must @ > 0;

^Email = String must @.Contains("@");

^Writable = ~Stream must @ is .Write;
```

## Type Guard Inheritance and Composition

* MAY inherit from other type guards by placing `: ^ParentGuard` prior to the `=` token (`^Identifier : ^Parent = TargetType must Expression;`).
* MUST implicitly forward the current type guard's base type to inherited parent type guards.
* MAY provide explicit generic arguments to parent type guards using `<@>` or concrete type identifiers.
* MUST evaluate all inherited parent `must` invariants in declaration order prior to evaluating the child type guard's own `must` expression.

```scatty
^Positive = @ must @ > 0;

^Big : ^Positive = @ must @ > 1000;

^Complex : ^Positive, ^AnotherComplex<@> = @ must @ % 2 == 0;
```

## Applying Type Guards

* MUST apply a type guard to a base type using the postfix `^` operator (`BaseType^Guard`).
* MUST define the physical memory layout solely via the `BaseType`.
* MUST evaluate the `^Guard` as a compile-time or runtime invariant.
* MUST implicitly pass the `BaseType` as the primary target type argument to the applied type guard.
* MAY apply a type guard directly to an instance method receiver (`@^Guard` or `ref @^Guard`).
* MUST require the caller's target expression to be statically typed or narrowed to `BaseType^Guard` when invoking a method declared with an `@^Guard` or `ref @^Guard` receiver.

```scatty
validAge: Int^Positive = 25 as ^;
email: String^Email = "test@domain.com" as ^;

ProcessAge(age: Int^Positive) { }

ProcessBigNumber(num: Float^Big) { }

Account 
{
    Balance: Int;
}

^Solvent = Account must @.Balance > 0;

Account with
{
    Withdraw(ref @^Solvent, amount: Int) 
    {
        @.Balance -=^ amount;
    }
}

rawAcc := Account { Balance: 100 };
rawAcc.Withdraw(50); // ERROR: 'rawAcc' is 'Account', not 'Account^Solvent'

may rawAcc is Account^Solvent 
{
    rawAcc.Withdraw(50); // Valid: 'rawAcc' is narrowed to 'Account^Solvent'
}
```

## Validation and Casting

* MAY implicitly downgrade a guarded type (`BaseType^Guard`) to its unguarded `BaseType` when passed by value.
* MUST NOT implicitly cast an unguarded `BaseType` to a guarded type (`BaseType^Guard`).
* MUST explicitly validate the invariant when casting from a base type to a guarded type, unless using `assume` (within an `unsafe` block or `unsafe` method) or a guarded assignment operator (`=^`).

### Operators

* MUST evaluate the type guard's `must` invariant at compile time whenever the operand of `as`, `as?`, `as!`, or `=^` is a compile-time constant expression:
  * If the compile-time evaluation yields `true`, the compiler MUST elide all runtime validation branches.
  * If the compile-time evaluation yields `false`, the compiler MUST reject the expression with a compile-time error (or evaluate `as?` directly to `Option::None`).
* MUST evaluate the invariant and implicitly narrow the type within conditional branches, `and` condition chains, and subsequent statements following an unconditional early-exit branch when using the `is` and `is not` operators.
* MUST validate the invariant at runtime (in both Debug and Release modes, unless proven at compile time) and throw a `CastException` upon failure when using the `as` operator.
* MUST validate the invariant at runtime and yield `Option<BaseType^Guard>` when using the `as?` operator.
* MUST validate the invariant at runtime (in both Debug and Release modes, unless proven at compile time) and immediately trigger runtime Abort or Fatal Panic mechanics upon failure when using the `as!` operator.
* MUST bypass runtime validation and force compiler acceptance when using the `assume` operator.
* MUST confine the use of the `assume` operator exclusively to within an `unsafe` block or `unsafe` method.
* MAY substitute the explicit target type with the contextual guard symbol (`^`) following `as`, `as?`, `as!`, or `assume` when the expected guarded type is statically known from context (explicitly typed declarations, assignments, method parameter types, `return` statements, or structural field initializers).
* MUST reject the contextual guard cast (`as ^`, `as? ^`, `as! ^`, `assume ^`) at compile time if the target context does not expect a guarded type or if the expected type is inferred (`:=`).
* MAY use a type guard pattern (`^Guard`) directly within a `match` arm to validate the invariant and narrow the matched subject within the arm's scope.

```scatty
rawValue: Int = -10;

may rawValue is Int^Positive 
{
    ProcessAge(rawValue);
}

// Throws 'CastException' if rawValue <= 0
valid := rawValue as Int^Positive;

// Aborts/Panics immediately if rawValue <= 0
forced := rawValue as! Int^Positive;

unsafe 
{
    // Unchecked cast (requires 'unsafe')
    uncheckedVal := rawValue assume Int^Positive;
}

// Compile-time evaluated: '11 > 0' is proven at compile time -> ZERO runtime branches
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

* MUST NOT mutate a direct or nested field or indexed element of a guarded variable (`instance: Type^Guard`) or guarded receiver (`ref @^Guard`) using standard assignment operators (`instance.Field = val`, `instance.Items[i] = val`).
* MUST require guarded assignment operators (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`) when mutating any direct or nested field or indexed element of a variable governed by a type guard (`instance.Field =^ val`, `instance.Sub.Field +=^ val`), re-evaluating all enclosing `^Guard` invariants along the mutated path from innermost to outermost after updating the target.
* MUST require guarded assignment operators (`@.Field =^ val`, `@.Field +=^ val`) when mutating any direct or nested field or indexed element of `@` within a method declaring a `ref @^Guard` receiver, re-evaluating the `^Guard` invariant according to the active build-mode validation rules.
* MUST automatically re-evaluate the `must` invariant of a guarded variable (`instance: Type^Guard`) or guarded receiver (`ref @^Guard`)—along with any intermediate guarded fields along the accessed path, from innermost to outermost—immediately after:
  * Invoking any mutating (`ref @`) method on `instance` (or `@`) or on any of its direct or nested fields or indexed elements (e.g., `list.Clear()` on `list: [Int]^NonEmpty`, or `user.Scores.Clear()` on `user: Profile^HasScores`), or
  * Passing `instance` (or `@`) or any of its direct or nested fields or indexed elements as a `ref` argument to any method parameter, regardless of whether the target parameter is typed as an unguarded base type or a guarded type.
* MUST also re-evaluate the `must` invariant of the guarded variable or guarded receiver (and any intermediate guarded fields along the path) on the error-return path if a mutating (`ref @`) method invoked on `instance` (or any of its direct or nested fields or indexed elements) or a method accepting `instance` (or any of its direct or nested fields or indexed elements) as a `ref` argument is declared with `throws` and returns an active exception.
* MUST enforce the following build-mode validation semantics for all statement-level guarded assignments (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`), guarded field mutations, and post-`ref` / post-`ref @` invariant checks (including on error-return paths) when the invariant cannot be statically proven at compile time:
  * In Debug mode, the compiler MUST evaluate the governing `must` invariant at runtime immediately after the mutation and trigger a Fatal Panic if the invariant evaluates to `false`.
  * In Release mode, the compiler MUST elide the runtime invariant check by default, executing the mutation with zero branching overhead.
  * Implementations MUST provide a compiler configuration flag that retains runtime invariant checks for all guarded mutations in Release mode, triggering an immediate process Abort upon invariant violation.
* MAY append the postfix `strict` modifier to any guarded variable declaration, guarded assignment (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`), guarded field mutation, or mutating `ref` / `ref @` invocation statement.
* MUST evaluate the runtime invariant check in both Debug and Release modes when the `strict` modifier is applied, triggering an immediate process Abort or Fatal Panic if the invariant evaluates to `false`.
* MUST reject applying the `strict` modifier to a statement that does not involve a guarded variable or guarded mutation.
* MUST always evaluate the runtime invariant check in both Debug and Release modes when `=^` is used inside a conditional assignment (`must`, `must!`, `may`, or `match`).

```scatty
User 
{
    Name: String;
    Age: Int;
    Scores: [Int];
}

Range 
{
    Min: Int^Positive;
    Max: Int^Positive;
}

^Adult      = User must @.Age >= 18;
^HasScores  = User must @.Scores.Count > 0;
^NonEmpty   = [Int] must @.Count > 0;
^ValidRange = Range must @.Min < @.Max;

ResetUser(ref u: User) 
{
    u.Age = 5;
}

SetHighValue(ref val: Int^Positive) 
{
    val =^ 1000 strict;
}

MutateExamples(
    ref age: Int^Positive,
    ref adult: User^Adult,
    ref player: User^HasScores,
    ref list: [Int]^NonEmpty,
    ref bounds: Range^ValidRange
) 
{
    // 1. Default guarded scalar mutation: checked in Debug, elided in default Release
    age -=^ 10;

    // 2. Strict guarded mutation: enforced at runtime in BOTH Debug and Release modes
    age -=^ 5 strict;
    adult.Age =^ 21 strict;

    // 3. Strict post-'ref @' and post-'ref' invariant checks on the root variable
    list.RemoveAt(0) strict;
    ResetUser(ref adult) strict;

    // 4. Post-'ref @' invariant check triggered by mutating a nested field ('player.Scores')
    player.Scores.Clear() strict;

    // 5. Post-'ref' invariant check on 'bounds' ('^ValidRange') triggered even though
    //    the parameter 'val' is itself guarded as 'Int^Positive'
    SetHighValue(ref bounds.Min) strict;
}
```
