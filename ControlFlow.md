# Control Flow

## Conditionals

* MUST evaluate a `Condition` expression to `Bool`.
* MUST interpret an unparenthesized `{` token in the header of an `if`, `for`, or `match` construct as the opening delimiter of the construct's body block.
* MUST enclose any structural initialization expression containing a `{ ... }` body (`StructInitBody`) in parentheses (`(Type { ... })`) when evaluated inside an `if` condition, `for` header, or `match` subject.

```C#
IsDebug :: true;

// Valid: '{' opens the 'if' block
if IsDebug 
{
    Console.WriteLine("Debug mode");
}

// Valid: struct initializer with '{ ... }' inside 'if' header is parenthesized
if currentConfig == (Config { Port: 8080 }) 
{
    Console.WriteLine("Default port");
}
```

### Conditional Binding

* MUST evaluate the right-hand expression of a conditional binding (`:=`) to an Option type (`T?`) or a weak reference type (`T weak`).
* MUST bind the unwrapped payload (or promoted strong reference `T`) to the new local identifier and continue evaluating the condition or execute the controlled block if the expression evaluates to `Option::Some` (or a live `T weak` with `strongCount > 0`).
* MUST short-circuit and skip execution of the controlled block if the expression evaluates to `Option::None` (or a disposed `T weak` with `strongCount == 0`).
* MAY chain multiple conditional bindings, conditional assignments, and boolean expressions within a single `Condition` using the `and` operator.
* MUST evaluate chained `and` clauses sequentially from left to right.
* MUST bring any local identifier introduced by a conditional binding (`:=`) into scope for all subsequent clauses to its right within the same `and` chain and within the positive branch of the `if` or `for` statement.
* MUST NOT access a local identifier introduced by a conditional binding within an `else` branch or outside the lexical scope of the positive branch.
* MUST NOT declare duplicate local identifiers across multiple conditional bindings within the same `Condition`.
* MUST NOT apply the `not` operator to a conditional binding or combine a conditional binding with the `or` operator.

```C#
maybeUser: User? = GetUser();

if user := maybeUser and session := GetSession(user.Id) and user.IsActive 
{
    Console.WriteLine(user.Name);
    Console.WriteLine(session.Token);
}
```

### Conditional Assignment

* MUST evaluate the right-hand expression to an Option type (`T?`) when using the standard assignment operator (`=`) within a condition clause.
* MUST resolve the left-hand `AssignablePath` to a mutable target of the unwrapped payload type `T`.
* MUST NOT use the `=` assignment operator with non-Option types within a condition clause.
* MAY use the guarded assignment operator (`=^`) within a condition clause when the left-hand `AssignablePath` resolves to a mutable target of a guarded type (`T^Guard`) and the right-hand expression evaluates to the underlying base type `T`.
* MUST evaluate `AssignablePath =^ Expression` within a condition clause as `AssignablePath = (Expression) as? T^Guard`.
* MAY chain conditional assignments with boolean expressions and conditional bindings using the `and` operator.
* MUST mutate the left-hand target with the unwrapped or validated payload and continue evaluating subsequent `and` clauses (or execute the controlled block) if the evaluation yields `Option::Some`.
* MUST skip the assignment, short-circuit any remaining `and` clauses, and transfer execution to the `else` branch if the evaluation yields `Option::None`.
* MUST NOT apply the `not` operator to a conditional assignment or combine a conditional assignment with the `or` operator.

```C#
SomeData 
{
    Value: Int;
    SafeAge: Int^Positive;

    UpdateFrom(ref @, rawAge: Int) 
    {
        if @.Value = rawAge as? Int^Positive and @.Value < 120 
        {
            Console.WriteLine("Value successfully updated!");
        }

        if @.SafeAge =^ rawAge - 1 
        {
            Console.WriteLine("SafeAge validated and updated!");
        }
    }
}
```

### Flow-Sensitive Type Narrowing

* MUST implicitly narrow the static type of a local variable, parameter, receiver (`@`), or immutable field path when evaluated with the `is` or `is not` operator against a variant subtype (`.Variant` or `Data::Variant`), a type guard (`^Guard` or `Type^Guard`), or a weak interface member (`.Member`).
* MUST apply positive type narrowing (`expr is Target`) to all subsequent clauses to the right within the same `and` condition chain and throughout the positive branch of the `if` or `for` statement.
* MUST apply negative type narrowing (`expr is not Target`) to the `else` branch of an `if` statement, refining `expr` to `Target` within the `else` scope.
* MUST apply early-exit type narrowing to all subsequent statements in the enclosing lexical scope whenever an `if` branch testing a negated condition (`expr is not Target`) unconditionally terminates execution of the current scope via `return`, `throw`, `break`, or `continue`.
* MUST narrow an expression of a tagged `Data` type to the single remaining variant subtype (`Data::Variant`) in the `else` branch or following an unconditional early-exit branch if all other declared variants of that `Data` type are excluded by `is` or `is not` checks.
* MUST immediately invalidate flow-sensitive type narrowing on a mutable variable or field at the point where it is reassigned, switched via `=::`, or passed as a `ref` argument to a method that may mutate its active variant or violate its type guard invariant.

```C#
ProcessOrder(order: Order, rawAge: Int) throws 
{
    // Narrowing across an 'and' chain
    if order is .Draft and order.Items.Count > 0 
    {
        Console.WriteLine(order.Items[0]);
    }

    // Early-exit narrowing via 'is not'
    if rawAge is not Int^Positive => return;

    // 'rawAge' is narrowed to 'Int^Positive' for the remainder of the method
    ProcessAge(rawAge);

    // Complementary variant narrowing after early exit on a two-variant Data type ('Order::Draft' | 'Order::Submitted')
    if order is .Draft => return;

    // 'order' is narrowed to 'Order::Submitted'
    Console.WriteLine(order.TransactionId);
}
```

## Loops

* MUST evaluate a single `Condition` expression as a `Bool` to perform conditional looping.
* MUST bind the iteration element to a new local variable (or unpack a tuple element into local variables via a destructuring pattern `(a, b)`) when using the `in` operator.
* MUST require the number of bindings in a loop destructuring pattern to match the exact element count of the iterated tuple, permitting `_` to discard individual elements.
* MUST evaluate the upper bound of a range expression (`..`) as exclusive, omitting the right-hand boundary value from the iteration sequence.
* MAY delegate the traversal strategy using the `by` modifier.
* MUST resolve the identifier provided to the `by` modifier to a type implementing the `~Iterator` interface.
* MAY use the `break` statement inside a `for` loop body to immediately terminate the innermost enclosing `for` loop.
* MAY use the `continue` statement inside a `for` loop body to immediately skip the remainder of the current iteration and advance to the next iteration of the innermost enclosing `for` loop.
* MUST NOT use `break` or `continue` outside the lexical body of a `for` loop (or across a lambda or `spawn` boundary).

```C#
// Conditional loop with break and continue
for x < 100 
{
    x += 10;
    if x == 50 => continue;
    if x > 80 => break;
}

// Conditional loop
for x < 100 
{
    x += 10;
}

// Iterator loop over an exclusive range (iterates 0 through 9)
for i in 0..10 
{
    Console.WriteLine(i);
}

// Iterator loop over a range with a step of 2 (0, 2, 4, 6, 8)
for i in 0..10..+2
{
    Console.WriteLine(i);
}

// Iterator loop over a reverse range (iterates 10 through 1)
for i in 10..0..-1
{
    Console.WriteLine(i);
}

// Default iteration
for item in list 
{
    Process(item);
}

// Tuple destructuring during map or pair iteration
for (key, score) in scores 
{
    Console.WriteLine("\(key): \(score)");
}

// Custom iteration strategy
for item in list by Parallel 
{
    Process(item);
}
```

## Pattern Matching

* MUST exhaustively cover all possible variants of the evaluated expression, or provide a wildcard `_` fallback.
* MAY bind the payload of a Variant Field to local identifiers using parentheses.
* MUST implicitly refine the matched expression to the evaluated variant or type guard within the match arm if payload binding is omitted.
* MUST evaluate every match arm to the exact same return type if the `match` construct is utilized as an expression.

### Type Guard Patterns and Where Clauses

* MAY specify a type guard pattern (`^Identifier` or `^Identifier<TypeArgs>`) as a match arm pattern when the type guard is applicable to the matched expression's type.
* MUST evaluate the type guard's `where` invariant against the matched expression at runtime and execute the arm if the invariant evaluates to `true`.
* MUST implicitly refine the type of the matched expression to `BaseType^Guard` within the body of a matching type guard arm.
* MAY attach a `where Expression` guard clause to a match arm following its pattern list.
* MUST evaluate the `where` guard expression to `Bool`.
* MUST bring any payload variables bound by the arm's pattern into scope within the `where` guard expression.
* MUST bind `@` within a match arm's `where` guard expression to the evaluated match subject.
* MUST bypass the match arm and continue evaluating subsequent arms if the pattern matches but its `where` guard evaluates to `false`.
* MUST NOT treat a match arm governed by a `^Guard` pattern or a `where` guard as exhaustive coverage for the underlying type or variant; an unguarded fallback arm or `_` wildcard MUST be provided in an exhaustive `match`.

```C#
^ClientError = Int where @ >= 400 and @ < 500;
^ServerError = Int where @ >= 500 and @ < 600;

match statusCode 
{
    200 => Console.WriteLine("OK");
    ^ClientError => Console.WriteLine("Client Error");
    ^ServerError => Console.WriteLine("Server Error");
    _ => Console.WriteLine("Other Status");
}

match shape 
{
    .Circle(r) where r > 100.0f => Console.WriteLine("Large Circle");
    .Circle(r) => Console.WriteLine("Standard Circle: \(r)");
    .Rectangle(w, h) where w == h => Console.WriteLine("Square");
    _ => Console.WriteLine("Other Shape");
}
```

### Execution Flow and Explicit Fallthrough

* MUST terminate execution of the `match` construct immediately after the first matching arm finishes (no implicit fallthrough).
* MAY use the `fall` statement within a block-bodied match arm to immediately transfer execution to the body of the next sequential arm without evaluating its pattern or `where` guard.
* MUST NOT use the `fall` statement if the `match` construct evaluates as an expression.
* MUST NOT use the `fall` statement to transfer control into a match arm that declares payload variable bindings.
* MUST NOT use the `fall` statement within the final match arm.

```C#
match statusCode 
{
    404 => Console.WriteLine("Not Found");
    
    // Cascading logic using 'fall'
    401, 403 
    {
        Console.WriteLine("Access Denied");
        fall; // Transfers control directly to the '_' block
    }
    
    _ => Console.WriteLine("Logging error...");
}

match some.VariedField 
{
    .Wow 
    {
        Console.WriteLine("Wow!");
        fall; // ERROR: Cannot fall into '.Hmm' because it binds 'x' and 'y'
    }
    
    .Hmm(x, y) 
    {
        Console.WriteLine(x + y);
    }
}

area := match shape.Type 
{
    .Circle(r) => Math.Pi * r * r;
    .Rectangle(w, h) => (w * h) as Float;
    _ => 0.0f;
};
```

### Non-Exhaustive Pattern Matching

* MUST NOT require exhaustive variant coverage or a wildcard `_` fallback when evaluating `match!` or `match?`.
* MUST immediately trigger runtime Abort or Fatal Panic mechanics if a `match!` construct fails to match the evaluated expression against any provided arm.
* MUST evaluate a `match?` construct used as an expression to `T?` (`Option<T>`), regardless of the number of match arms.
* MUST implicitly wrap the evaluated value of a matching `match?` arm into `Option::Some`.
* MUST yield `Option::None` if a `match?` expression fails to match any provided arm (or perform a no-op if `match?` is used as a void statement).

```C#
// 'match!' triggers Abort/Panic if 'shape.Type' is not a 'Circle'
match! shape.Type  
{
    .Circle(r) => Console.WriteLine("Wow!");
}

// 'area' is 'Float?' ('Option::Some' if 'Circle' or 'Rectangle', 'Option::None' otherwise)
area := match? shape.Type 
{
    .Circle(r) => Math.Pi * r * r;
    .Rectangle(w, h) => (w * h) as Float;
};

// 'circleArea' is 'Float?' ('Option::Some' if 'Circle', 'Option::None' otherwise)
circleArea := match? shape.Type 
{
    .Circle(r) => Math.Pi * r * r;
};
```
