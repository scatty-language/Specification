# Control Flow

## Gates and Conditionals

* MUST evaluate a `Condition` expression to `Bool`.
* MUST interpret an unparenthesized `{` token in the header of a `may`, `in`, or `match` construct as the opening delimiter of the construct's body block.
* MUST enclose any structural initialization expression containing a `{ ... }` body (`StructInitBody`) in parentheses (`(Type { ... })`) when evaluated inside a `must`, `must!`, `may`, `in`, or `match` header.

### Filter and Guard Gates

* MAY declare a standalone gate (`must Condition;` or `must { Condition; ... }`) inside an `in` pipeline block or inside a method that returns no value (void).
* MUST immediately skip all subsequent statements in the `in` block for the current element and advance the innermost active stage if any condition in a standalone `must` gate inside an `in` pipeline evaluates to `false` or `Option::None`.
* MUST immediately return from the enclosing method (`return;`) if any condition in a standalone `must` gate inside a void method (outside an `in` pipeline) evaluates to `false` or `Option::None`.
* MUST reject a standalone `must` gate lacking an `or` fallback clause inside a value-returning method (outside an `in` pipeline) at compile time.
* MAY attach an `or` fallback clause to a `must` gate (`must Condition or Fallback;` or `must { Condition; ... } or Fallback;`) in any method or pipeline block.
* MUST require the `or` fallback clause of a `must` gate to unconditionally exit the enclosing scope via `return`, `throw`, `break`, or `continue`.
* MUST evaluate multiple conditions inside a `must { ... }` block sequentially from top to bottom and short-circuit immediately upon the first condition that evaluates to `false` or `Option::None`.
* MUST bring all local identifiers bound via `:=` and all type narrowings proven via `is` within a `must` gate into the enclosing lexical scope for all subsequent statements below the gate.

```C#
// 1. Standalone 'must' gate in a void method:
// Automatically returns if 'user' is None or inactive
SendNotification(userId: Int) 
{
    must 
    {
        user := FindUser(userId);
        user.IsActive;
    }

    Notify(user);
}

// 2. 'must ... or' gate in a value-returning method:
ProcessBatch(orders: List<Order>, userId: Int) -> Float 
{
    must 
    {
        orders.Count > 0;
        user := FindUser(userId);
        user.IsActive;
    } 
    or return 0.0f;

    order in orders 
    {
        must 
        {
            order is .Paid;
            order.Amount > 0.0f;
            order.OwnerId == user.Id;
        }

        out sum total := order.Amount;
    }

    return total;
}
```

### Invariant Gates

* MAY declare a critical invariant gate using `must! Condition;` or `must! { Condition; ... }`.
* MUST evaluate the conditions of a `must!` gate in order from top to bottom in both Debug and Release modes, unless statically proven at compile time.
* MUST immediately trigger runtime Abort or Fatal Panic mechanics if any condition in a `must!` gate evaluates to `false` or `Option::None` (or a disposed `T weak` with `strongCount == 0`).
* MUST reject a `must!` gate at compile time if any condition is a compile-time constant expression that evaluates to `false` or `Option::None`.
* MUST bring all local identifiers bound via `:=` and all type narrowings proven via `is` within a `must!` gate into the enclosing lexical scope for all subsequent statements below the gate.

```C#
ExecuteCritical(rawUser: User?, rawAge: Int) 
{
    must! 
    {
        user := rawUser;
        rawAge is Int^Positive;
        user.IsVerified;
    }

    RunPrivilegedAction(user, rawAge);
}
```

### Conditional Branches

* MAY execute a conditional branch using `may Condition { ... }` or `may Condition => Statement;`.
* MAY chain fallback branches onto a `may` statement using `or may Condition ...` or a terminal fallback `or { ... }`, `or => Statement;`, or `or Statement;`.
* MUST execute the positive branch of `may` if `Condition` evaluates to `true` (or `Option::Some`), or transfer control to the subsequent `or` branch (if present) if `Condition` evaluates to `false` (or `Option::None`).
* MUST continue execution to the statement immediately following the `may` construct unless the executed branch explicitly exits the scope.

```C#
must x > 0;
must x > 100;

may x == 101 => Console.WriteLine("101!");
or Console.WriteLine("Not 101");

item in items 
{
    must item.IsValid;
    may item.IsRare => LogRareItem(item);
    must! item.Price >= 0.0f;

    out sum total := item.Price;
}
```

### Conditional Binding

* MUST evaluate the right-hand expression of a conditional binding (`:=`) to an Option type (`T?`) or a weak reference type (`T weak`).
* MUST bind the unwrapped payload (or promoted strong reference `T`) to the new local identifier and continue evaluating the condition if the expression evaluates to `Option::Some` (or a live `T weak` with `strongCount > 0`).
* MUST short-circuit the condition if the expression evaluates to `Option::None` (or a disposed `T weak` with `strongCount == 0`).
* MAY chain multiple conditional bindings, conditional assignments, and boolean expressions using the `and` operator or across sequential lines inside a `must { ... }` or `must! { ... }` block.
* MUST bring any local identifier introduced by a conditional binding (`:=`) into scope for all subsequent clauses to its right within an `and` chain, for all subsequent lines within a `must { ... }` or `must! { ... }` block, and:
  * within the positive branch of a `may` statement,
  * within the matching arm body of a subjectless `match { ... }` construct, or
  * within the enclosing lexical scope for all statements following a `must` or `must!` gate.
* MUST NOT access a local identifier introduced by a conditional binding inside an `or` fallback branch.
* MUST NOT declare duplicate local identifiers across multiple conditional bindings within the same condition or `must` block.
* MUST NOT apply the `not` operator to a conditional binding or combine a conditional binding with the boolean `or` operator.

### Conditional Assignment

* MUST evaluate the right-hand expression to an Option type (`T?`) when using the standard assignment operator (`=`) within a condition clause.
* MUST resolve the left-hand `AssignablePath` to a mutable target of the unwrapped payload type `T`.
* MUST NOT use the `=` assignment operator with non-Option types within a condition clause.
* MAY use the guarded assignment operator (`=^`) within a condition clause when the left-hand `AssignablePath` resolves to a mutable target of a guarded type (`T^Guard`) and the right-hand expression evaluates to the underlying base type `T`.
* MUST evaluate `AssignablePath =^ Expression` within a condition clause as `AssignablePath = (Expression) as? T^Guard`.
* MAY chain conditional assignments with boolean expressions and conditional bindings using `and` or inside a `must { ... }` or `must! { ... }` block.
* MUST mutate the left-hand target with the unwrapped or validated payload and continue evaluating subsequent clauses if the evaluation yields `Option::Some`.
* MUST skip the assignment and short-circuit remaining clauses if the evaluation yields `Option::None`.
* MUST NOT apply the `not` operator to a conditional assignment or combine a conditional assignment with the boolean `or` operator.

```C#
SomeData 
{
    Value: Int;
    SafeAge: Int^Positive;

    UpdateFrom(ref @, rawAge: Int) 
    {
        may @.Value = rawAge as? Int^Positive and @.Value < 120 
        {
            Console.WriteLine("Value successfully updated!");
        }

        may @.SafeAge =^ rawAge - 1 
        {
            Console.WriteLine("SafeAge validated and updated!");
        }
    }
}
```

### Flow-Sensitive Type Narrowing

* MUST implicitly narrow the static type of a local variable, parameter, receiver (`@`), or immutable field path when evaluated with the `is` or `is not` operator against a variant subtype (`.Variant` or `Data::Variant`), a type guard (`^Guard` or `Type^Guard`), a structural pattern (`{ ... }`), a tuple pattern (`( ... )`), or a weak interface member (`.Member`).
* MUST bring any local identifier (`local_id`) bound within an `expr is Pattern` clause into scope for all subsequent clauses to its right within an `and` chain, for all subsequent lines within a `must { ... }` or `must! { ... }` block, throughout the positive branch of a `may` statement, and for all subsequent statements in the enclosing lexical scope following a `must` or `must!` gate.
* MUST NOT declare local variable bindings inside an `is not` expression or combine a binding `is` expression with the boolean `or` operator.
* MUST apply positive type narrowing (`expr is Target`) to all subsequent clauses within the same `and` chain or `must { ... }` / `must! { ... }` block, throughout the positive branch of a `may` statement, and to all statements in the enclosing scope following a `must` or `must!` gate.
* MUST apply negative type narrowing (`expr is not Target`) to the `or` branch of a `may` statement, refining `expr` to `Target` within the `or` scope.
* MUST apply type narrowing to all subsequent statements in the enclosing lexical scope whenever a `may` branch testing a negated condition (`may expr is not Target => return;`) unconditionally terminates execution of the current scope via `return`, `throw`, `break`, or `continue`.
* MUST narrow an expression of a tagged `Data` type to the single remaining variant subtype (`Data::Variant`) in the `or` branch or following an unconditional early-exit branch or `must` gate if all other declared variants of that `Data` type are excluded.
* MUST immediately invalidate flow-sensitive type narrowing on a mutable variable or field at the point where it is reassigned, switched via `=::`, or passed as a `ref` argument to a method that may mutate its active variant or violate its type guard invariant.

## Staged Iteration

* MAY declare a staged iteration pipeline over a collection or range using `item in iterable { ... }` or `item in iterable => Statement;` without a leading loop keyword.
* MAY prefix the stage binding with `ref` (`ref item in collection { ... }`) to mutate collection elements in-place.
* MAY unpack tuple elements in the stage header using a destructuring pattern (`(a, b) in iterable { ... }`), requiring the binding count to match the tuple element count and permitting `_` to discard elements.
* MUST evaluate the upper bound of a range expression (`start..end` or `start..end..step`) as exclusive.
* MAY delegate the execution schedule of a pipeline using the `by` modifier (`item in iterable by Schedule { ... }`), resolving the identifier to a type implementing the `~Iterator` interface.
* MAY omit both the stage binding and the iterable (`in { ... }`) to declare an unbounded loop that executes repeatedly until terminated via `break`, `return`, or `throw`.
* MAY use `break` inside an `in` block to immediately terminate the entire pipeline.
* MAY use `continue` inside an `in` block to immediately skip the remainder of the current iteration and advance the innermost active stage.
* MUST NOT use `break` or `continue` outside the lexical body of an `in` block or across a lambda or `spawn` boundary.

```C#
// Range iteration (0 through 9)
i in 0..10 
{
    Console.WriteLine(i);
}

// Reverse range with step (10 down to 1)
i in 10..0..-1 
{
    Console.WriteLine(i);
}

// In-place mutation of collection elements
ref enemy in enemies 
{
    must enemy.Hp > 0;
    enemy.Hp -= 10;
}

// Unbounded conditional loop using 'in { ... }'
in 
{
    must x < 100 or break;
    x += 10;
}
```

### Filter Stages and Flattened Inner Stages

* MAY place one or more filter gates (`must Condition;` or `must { Condition; ... }`) at any point inside an `in` pipeline block.
* MUST immediately skip all subsequent statements in the `in` block for the current element and advance the innermost active stage if a `must` gate evaluates to `false` or `Option::None`.
* MUST bring variables bound via `:=` and type narrowings proven via `is` inside a `must` gate into scope for all subsequent statements below that gate within the `in` block.
* MAY place one or more stage expansion statements (`inner in iterable;`) inside an `in` block to flatten nested collections without nested blocks.
* MUST iterate over the expanded `iterable` for each upstream element that reaches the `inner in iterable;` stage, bringing `inner` into scope for all subsequent statements in the `in` block.

```C#
dept in company.Departments 
{
    must not dept.IsArchived;

    emp in dept.Employees;
    must emp.Role is .Engineer;

    task in emp.Tasks;
    must task.IsOverdue;

    Console.WriteLine("\(dept.Name): \(emp.Name) -> \(task.Title)");
}
```

### Output Stages and Multi-Sink Aggregation

* MAY declare one or more named output sinks (`out [Reducer] local_id ...`) inside a statement-level `in` pipeline block.
* MUST treat `count`, `sum`, `min`, `max`, `all`, `any`, `first`, and `last` as contextual reducer modifiers when placed immediately after the `out` keyword.
* MUST hoist and introduce every variable declared via a named `out` statement into both:
  * the remainder of the current `in` block immediately following the `out` statement, and
  * the lexical scope immediately enclosing the `in` pipeline statement after the `in` block completes.
* MAY read the current accumulated value of a named `out` binding in any subsequent statement or `must` gate positioned below that `out` statement within the same `in` block.
* MUST NOT reassign or mutate a named `out` binding outside of its `out` statement while inside the lexical body of the `in` block.
* MUST initialize and update named `out` bindings according to the specified reducer:
  * `out count local_id` (inferred as `Int` if `: Type` is omitted): Initializes `local_id` to `0` prior to the pipeline and increments it by `1` each time the statement is reached.
  * `out sum local_id := Expression` (inferred as the numeric type `T` of `Expression`): Initializes `local_id` to the additive identity (`0` or `0.0`) prior to the pipeline and adds the evaluated `Expression` each time the statement is reached.
  * `out min local_id := Expression` (inferred as `T?` where `T` is the `~Comparable` type of `Expression`): Initializes `local_id` to `Option::None` prior to the pipeline and updates it to `Option::Some` containing the minimum evaluated value according to `~Comparable`.
  * `out max local_id := Expression` (inferred as `T?` where `T` is the `~Comparable` type of `Expression`): Initializes `local_id` to `Option::None` prior to the pipeline and updates it to `Option::Some` containing the maximum evaluated value according to `~Comparable`.
  * `out all local_id := Expression` (requires `Expression` of type `Bool`; inferred as `Bool`): Initializes `local_id` to `true` prior to the pipeline and updates it to `local_id and Expression` each time the statement is reached.
  * `out any local_id := Expression` (requires `Expression` of type `Bool`; inferred as `Bool`): Initializes `local_id` to `false` prior to the pipeline and updates it to `local_id or Expression` each time the statement is reached.
  * `out first local_id := Expression` (inferred as `T?` where `T` is the type of `Expression`): Initializes `local_id` to `Option::None` prior to the pipeline and assigns `Option::Some(Expression)` only upon the first element reaching the statement.
  * `out last local_id := Expression` (inferred as `T?` where `T` is the type of `Expression`): Initializes `local_id` to `Option::None` prior to the pipeline and overwrites it with `Option::Some(Expression)` each time the statement is reached.
  * `out local_id: CollectionType = Expression` or `out local_id := Expression` (without a reducer modifier): Initializes an empty collection prior to the pipeline (defaulting to `[+ T]` when inferred via `:=`) and appends or inserts the evaluated `Expression` each time the statement is reached.
* MAY attach a trailing `must Condition` filter to any `out` statement to conditionally gate that specific output sink without halting subsequent statements in the `in` block.
* MUST automatically short-circuit and terminate the `in` pipeline early if the `in` block contains no side-effecting statements and all declared `out` sinks have reached a terminal state (`out first` has captured a value, `out all` has evaluated to `false`, or `out any` has evaluated to `true`).
* MAY evaluate an `in` pipeline as an expression using anonymous `out [Reducer] [Expression] [must Condition];` statements:
  * If no reducer is specified and the expected type is a collection, the pipeline MUST allocate the target collection, append or insert each emitted `out` value, and evaluate to that collection.
  * If a reducer (`count`, `sum`, `min`, `max`, `all`, `any`, `first`, `last`) is specified, the pipeline MUST evaluate to the reduced scalar or `Option` result.
  * If no reducer is specified and the expected type is `T?` (`Option<T>`), the pipeline MUST wrap the first emitted `out` value into `Option::Some`, immediately terminate the pipeline, and return it (or return `Option::None` if no element reaches `out`).
* MUST NOT mix anonymous `out` statements and named `out` statements within the same `in` pipeline block.

```C#
order in orders by Parallel 
{
    must order is .Paid and order.Amount > 0.0f;

    net := order.Amount * (1.0f - order.Discount);

    out count paidCount;
    out sum   totalRevenue := net;
    out max   peakOrder    := net;
    out all   allVerified  := order.IsVerified;
    out any   hasFraud     := order.Risk > 90;
    out vipOrders: [+ Int] = order.Id must net > 1000.0f;
}

item in cargoItems 
{
    out sum currentWeight := item.Weight;
    must currentWeight <= 50.0f or break;

    out loadedItems := item;
}
```

## Pattern Matching

* MAY omit the subject expression in a `match` construct (`match { ... }`) to evaluate a sequential table of boolean `Condition` arms.
* MUST evaluate boolean arms in a subjectless `match { ... }` sequentially from top to bottom and execute the first arm whose condition evaluates to `true`, or the `_` fallback arm.
* MUST exhaustively cover all possible variants of the evaluated expression, or provide a wildcard `_` fallback.
* MAY match the subject expression against variant patterns (`.Variant`), structural patterns (`{ Field: sub_pattern }`), tuple patterns (`(sub_pattern, sub_pattern)`), type guard patterns (`^Guard`), literals, or the wildcard `_`.
* MAY nest patterns recursively within variant payload patterns, structural field patterns, and tuple patterns.
* MAY bind any matched variant payload field, structural field, or tuple element to a local variable (`local_id`) or discard it using `_`.
* MUST require a positional variant pattern (without explicit field identifiers) to match the exact payload field count of the variant, using `_` to discard unused fields.
* MAY match variant payload fields by explicit field identifier (`FieldName: sub_pattern`) and omit remaining payload fields when named pattern arguments are used.
* MAY match structural or shared fields of a `Data` type by field identifier using `{ FieldName: sub_pattern }`, either standalone or following a variant pattern (`.Variant(...) { FieldName: sub_pattern }`), and omit remaining fields.
* MAY specify multiple comma-separated patterns (`pattern1, pattern2`) on a single match arm.
* MUST require every pattern in a comma-separated pattern list (`pattern1, pattern2`) to bind the exact same set of `local_id` identifiers with identical static types if any local variable bindings are declared.
* MUST implicitly refine the matched expression and any matched fields to the evaluated variant subtype or type guard within the match arm.
* MUST evaluate every match arm to the exact same return type if the `match` construct is utilized as an expression.

```C#
match response 
{
    // Positional variant pattern with literal and '_' discard
    .Error(404, msg, _) => HandleNotFound(msg);

    // Named variant payload pattern (omits remaining payload fields) with nested guard
    .Error(Code: ^ServerError, Message: msg) => HandleServerError(response.Code, msg);

    // Nested variant pattern and shared structural field pattern
    .Ok(.User(admin)) { Endpoint: "localhost" } => ProcessLocalAdmin(admin);

    // Structural pattern matching shared fields across any variant
    { Endpoint: "deprecated" } => RejectEndpoint();

    // Multi-pattern arm binding the same 'id' identifier across both variants
    .Created(id), .Updated(id) => LogChange(id);

    _ => Console.WriteLine("Unhandled response");
}

match (request.Method, user.IsAuthenticated) 
{
    (.Get, _) => AllowRead();
    (.Post, true), (.Put, true) => AllowWrite();
    _ => DenyAccess();
}
```

### Type Guard Patterns and `and` Clauses

* MAY specify a type guard pattern (`^Identifier` or `^Identifier<TypeArgs>`) as a match arm pattern or nested sub-pattern when the type guard is applicable to the target type.
* MUST evaluate the type guard's `must` invariant against the target value at runtime and match if the invariant evaluates to `true`.
* MUST implicitly refine the static type of the matched expression or field to `BaseType^Guard` within the body of a matching arm.
* MAY attach an `and Condition` clause to a match arm following its pattern list.
* MUST evaluate the `and` condition to `Bool`.
* MUST bring any variables bound by the arm's pattern into scope within the `and` condition.
* MUST bind `@` within a match arm's `and` condition to the evaluated match subject.
* MUST bypass the match arm and continue evaluating subsequent arms if the pattern matches but its `and` condition evaluates to `false`.
* MUST NOT treat a match arm governed by a `^Guard` pattern, a nested literal or structural field pattern, or an `and` condition as exhaustive coverage for the underlying type or variant; an unguarded fallback arm or `_` wildcard MUST be provided in an exhaustive `match`.

```C#
^ClientError = Int must @ >= 400 and @ < 500;
^ServerError = Int must @ >= 500 and @ < 600;

match statusCode 
{
    200 => Console.WriteLine("OK");
    ^ClientError => Console.WriteLine("Client Error");
    ^ServerError => Console.WriteLine("Server Error");
    _ => Console.WriteLine("Other Status");
}

match shape 
{
    .Circle(r) and r > 100.0f => Console.WriteLine("Large Circle");
    .Circle(r) => Console.WriteLine("Standard Circle: \(r)");
    .Rectangle(w, h) and w == h => Console.WriteLine("Square");
    _ => Console.WriteLine("Other Shape");
}
```

### Execution Flow and Explicit Fallthrough

* MUST terminate execution of the `match` construct immediately after the first matching arm finishes (no implicit fallthrough).
* MAY use the `fall` statement within a block-bodied match arm to immediately transfer execution to the body of the next sequential arm without evaluating its pattern or `and` condition.
* MUST NOT use the `fall` statement if the `match` construct evaluates as an expression.
* MUST NOT use the `fall` statement to transfer control into a match arm that declares payload variable bindings.
* MUST NOT use the `fall` statement within the final match arm.

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
