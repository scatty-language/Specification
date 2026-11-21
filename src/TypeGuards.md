# Type Guards

* MUST declare a type guard at the file's top level using the prefix `^` operator (`^Identifier = TargetType where Expression;`).
* MUST NOT declare a type guard within a `Data` declaration, an `~Interface` declaration, or a `with` extension block.
* MUST evaluate the `where` clause expression to a pure, side-effect-free `Bool`.
* MUST compile each type guard `where` clause as a non-throwing predicate.
* MAY invoke `throws` methods (directly or via `try?` or `try!`) within a `where` clause expression without a compile-time error.
* MUST immediately trigger runtime Abort or Fatal Panic mechanics if an uncaught error is thrown during the evaluation of a `where` clause.
* MUST explicitly specify the base type being guarded immediately following the `=` token.
* MAY specify a `Data` type, an `~Interface` type, a generic type parameter, or `@` as the `TargetType` of a type guard declaration.
* MAY specify `@` as the `TargetType` to create a universal type guard applicable to any type supporting the operations in the `where` expression.
* MUST evaluate `@` within the `where` expression as the immutable instance receiver.
* MAY declare explicit generic parameters and assign a type parameter as the `TargetType`.
* MAY omit the base type and use the contextual guard pattern (`is ^Guard` or `is ^Guard<TypeArgs>`) when the type guard is applicable to the left-hand expression's static type.

```scatty
^Positive = @ where @ > 0;

^PositiveNumber<T: ~INumber> = T where @ > 0;

^Email = String where @.Contains("@");

^Writable = ~Stream where @ is .Write;

Validator
{
    CheckStrict(val: Int) throws -> Bool
    {
        may val < 0 => throw Error.New("Negative");
        return val > 0;
    }

    TryValidate(val: Int) -> Bool
    {
        return (try? Validator.CheckStrict(val)) ?? false;
    }
}

// Valid: compiled as a non-throwing predicate; triggers Abort/Panic if 'CheckStrict' throws
^StrictPositive = Int where Validator.CheckStrict(@);

// Valid: 'TryValidate' catches internal errors via 'try?' and returns 'Bool'
^SafePositive = Int where Validator.TryValidate(@);
```

## Type Guard Inheritance and Composition

* MAY inherit from other type guards by placing `: ^ParentGuard` prior to the `=` token (`^Identifier : ^Parent = TargetType where Expression;`).
* MUST implicitly forward the current type guard's base type to inherited parent type guards.
* MAY provide explicit generic arguments to parent type guards using `<@>` or concrete type identifiers.
* MUST evaluate all inherited parent `where` invariants in declaration order prior to evaluating the child type guard's own `where` expression.

```scatty
^Positive = @ where @ > 0;

^Big : ^Positive = @ where @ > 1000;

^Complex : ^Positive, ^AnotherComplex<@> = @ where @ % 2 == 0;
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

^Solvent = Account where @.Balance > 0;

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

* MUST evaluate the type guard's `where` invariant at compile time whenever the operand of `as`, `as?`, `as!`, or `=^` is a compile-time constant expression:
  * If the compile-time evaluation yields `true`, the compiler MUST elide all runtime validation branches.
  * If the compile-time evaluation yields `false`, the compiler MUST reject the expression with a compile-time error (or evaluate `as?` directly to `Option::None`).
* MUST evaluate the invariant and implicitly narrow the type within conditional branches, `and` condition chains, and subsequent statements following an unconditional early-exit branch when using the `is` and `is not` operators.
* MUST validate the invariant at runtime (in both Debug and Release modes, unless proven at compile time) and throw `CastError` upon failure when using the `as` operator.
* MUST validate the invariant at runtime (in both Debug and Release modes, unless proven at compile time) and immediately trigger runtime Abort or Fatal Panic mechanics (bypassing any enclosing `catch` handlers) upon failure when using the `as!` operator.
* MUST validate the invariant at runtime and yield `Option<BaseType^Guard>` when using the `as?` operator.
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

// Throws 'CastError' if rawValue <= 0
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
    .Some(val) => ProcessAge(val);
    .None => Console.WriteLine("Invalid!");
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

* MUST classify statement-level guarded variable declarations (`=^`), guarded assignments (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`), guarded field mutations, post-`ref` / post-`ref @` invariant re-evaluations, and post-`ref`-stage loop exit checks as non-throwing operations.
* MUST NOT mutate a direct or nested field or indexed element of a guarded variable (`instance: Type^Guard`) or guarded receiver (`ref @^Guard`) using standard assignment operators (`instance.Field = val`, `instance.Items[i] = val`).
* MUST require guarded assignment operators (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`) when mutating any direct or nested field or indexed element of a variable governed by a type guard (`instance.Field =^ val`, `instance.Sub.Field +=^ val`), re-evaluating all enclosing `^Guard` invariants along the mutated path from innermost to outermost after updating the target.
* MUST require guarded assignment operators (`@.Field =^ val`, `@.Field +=^ val`) when mutating any direct or nested field or indexed element of `@` within a method declaring a `ref @^Guard` receiver, re-evaluating the `^Guard` invariant after updating the target.
* MUST automatically re-evaluate the `where` invariant of a guarded variable (`instance: Type^Guard`) or guarded receiver (`ref @^Guard`)—along with any intermediate guarded fields along the accessed path, from innermost to outermost—immediately after:
  * Invoking any mutating (`ref @`) method on `instance` (or `@`) or on any of its direct or nested fields or indexed elements (e.g., `list.Clear()` on `list: [Int]^NonEmpty`, or `user.Scores.Clear()` on `user: Profile^HasScores`), or
  * Passing `instance` (or `@`) or any of its direct or nested fields or indexed elements as a `ref` argument to any method parameter, regardless of whether the target parameter is typed as an unguarded base type or a guarded type.
* MUST evaluate the governing `where` invariant at runtime in both Debug and Release modes for all statement-level guarded variable declarations (`=^`), guarded assignments (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`), guarded field mutations, and post-`ref` / post-`ref @` invariant checks whenever the invariant is not statically proven at compile time.
* MUST immediately trigger runtime Abort or Fatal Panic mechanics (bypassing all `catch` handlers) if any governing `where` invariant evaluates to `false`.
* MUST re-evaluate the `where` invariant of the guarded variable or guarded receiver (and any intermediate guarded fields along the path) on the error-return path in both Debug and Release modes if a mutating (`ref @`) method invoked on `instance` (or any of its direct or nested fields or indexed elements) or a method accepting `instance` (or any of its direct or nested fields or indexed elements) as a `ref` argument is declared with `throws` and returns an active error:
  1. Preserve the active propagating error in a frame-local cleanup save slot (clearing the active error slot indicator and TLS error slot) prior to evaluating the `where` invariant.
  2. If all enclosing `where` invariants evaluate to `true`, restore the saved error into the TLS error slot and continue error propagation.
  3. If any enclosing `where` invariant evaluates to `false`, immediately trigger runtime Abort or Fatal Panic mechanics (bypassing all `catch` handlers).
* MUST evaluate `=^` inside a conditional assignment (`where`, `must`, `must!`, `may`, or `match`) in both Debug and Release modes as `AssignablePath = (Expression) as? T^Guard`, skipping the assignment without aborting if the `where` invariant evaluates to `false`.
* MUST treat `item` as a mutable guarded target of type `T^Guard` inside a `ref item in collection` loop when the element type of `collection` is guarded (`[T^Guard]`), requiring guarded assignment operators (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`) and element-level post-`ref` / post-`ref @` invariant checks for mutations of `item`.
* MUST permit standard (non-guarded) mutation operators on `item` inside a `ref item in collection` loop when the element type of `collection` is unguarded, even if `collection` itself or an enclosing container along its path is governed by a type guard (`list: [Int]^AllPositive` or `user.Scores` on `user: User^HasScores`).
* MUST automatically re-evaluate all enclosing `^Guard` invariants along `collection`'s path (from innermost to outermost) once upon exiting a `ref item in collection` loop (via normal completion, `break`, `return`, or error propagation) in both Debug and Release modes, unless statically proven at compile time, immediately triggering runtime Abort or Fatal Panic mechanics if any invariant evaluates to `false`.

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

^Adult      = User where @.Age >= 18;
^HasScores  = User where @.Scores.Count > 0;
^NonEmpty   = [Int] where @.Count > 0;
^ValidRange = Range where @.Min < @.Max;

ResetUser(ref u: User)
{
    u.Age = 5;
}

SetHighValue(ref val: Int^Positive)
{
    val =^ 1000;
}

MutateExamples(
    ref age: Int^Positive,
    ref adult: User^Adult,
    ref player: User^HasScores,
    ref list: [Int]^NonEmpty,
    ref bounds: Range^ValidRange
)
{
    // 1. Guarded scalar and field mutations: checked in both Debug and Release (Aborts/Panics on failure)
    age -=^ 5;
    adult.Age =^ 21;

    // 2. Post-'ref @' and post-'ref' invariant checks on the root variable
    list.RemoveAt(0);
    ResetUser(ref adult);

    // 3. Post-'ref @' invariant check triggered by mutating a nested field ('player.Scores')
    player.Scores.Clear();

    // 4. Post-'ref' invariant check on 'bounds' ('^ValidRange') triggered even though
    //    the parameter 'val' is itself guarded as 'Int^Positive'
    SetHighValue(ref bounds.Min);

    // 5. Non-aborting conditional guarded update via 'may'
    may age =^ age - 10
    {
        Console.WriteLine("Updated age: \(age)");
    }
}

MutateGuardedCollections(
    ref positiveScores: [Int^Positive],
    ref nonEmptyList: [Int]^NonEmpty,
    ref player: User^HasScores
)
{
    // Guarded element type ('Int^Positive'): 's' requires guarded assignment ('-=^')
    ref s in positiveScores
    {
        s -=^ 1;
    }

    // Guarded collection ('[Int]^NonEmpty'): standard mutation inside; checked once on exit
    ref n in nonEmptyList
    {
        n += 10;
    }

    // Guarded container ('User^HasScores'): 'player.Scores' checked against '^HasScores' once on exit
    ref sc in player.Scores
    {
        sc *= 2;
    }
}
```
