# Control Flow

## Conditionals

* MUST evaluate a `Condition` expression to `Bool`.
* MUST interpret an unparenthesized `{` token in the header of a `where`, `in`, or `match` construct as the opening delimiter of the construct's body block.
* MUST enclose any structural initialization expression containing a `{ ... }` body (`StructInitBody`) in parentheses (`(Type { ... })`) when evaluated inside a `where` condition, `in` header, or `match` header.
* MAY execute a conditional branch using `where Condition { ... }` or `where Condition => Statement;`, optionally followed by an `else` branch.
* MAY declare a guard-exit statement using `where Condition else FallbackStatement;` without a positive branch body.
* MUST require the `else` branch of a guard-exit statement (`where Condition else FallbackStatement;`) to unconditionally exit the enclosing scope via `return`, `throw`, `break`, or `continue`.
* MAY declare a standalone filter gate (`where Condition;`) exclusively within the lexical body of an `in` pipeline block.
* MUST reject a standalone `where Condition;` statement (lacking both a positive body and an `else` branch) outside of an `in` pipeline block at compile time.

```C#
IsDebug :: true;

// Conditional execution with '=>' or block '{}'
where IsDebug => Console.WriteLine("Debug mode");

where currentConfig == (Config { Port: 8080 }) 
{
    Console.WriteLine("Default port");
}

// Guard-exit statement: exits method immediately if condition fails
where orders.Count > 0 else return 0;
```

### Conditional Binding

* MUST evaluate the right-hand expression of a conditional binding (`:=`) to an Option type (`T?`) or a weak reference type (`T weak`).
* MUST bind the unwrapped payload (or promoted strong reference `T`) to the new local identifier and continue evaluating the condition if the expression evaluates to `Option::Some` (or a live `T weak` with `strongCount > 0`).
* MUST short-circuit the condition and skip the positive branch (or execute the `else` branch) if the expression evaluates to `Option::None` (or a disposed `T weak` with `strongCount == 0`).
* MAY chain multiple conditional bindings, conditional assignments, and boolean expressions within a single `Condition` using the `and` operator.
* MUST evaluate chained `and` clauses sequentially from left to right.
* MUST bring any local identifier introduced by a conditional binding (`:=`) into scope for all subsequent clauses to its right within the same `and` chain and within the positive branch of the `where` statement.
* MUST bring any local identifier introduced by a conditional binding (`:=`) into the enclosing lexical scope for all subsequent statements when used in a guard-exit statement (`where user := maybeUser else return;`) or a stage filter (`where user := maybeUser;`).
* MUST NOT access a local identifier introduced by a conditional binding within an `else` branch.
* MUST NOT declare duplicate local identifiers across multiple conditional bindings within the same `Condition`.
* MUST NOT apply the `not` operator to a conditional binding or combine a conditional binding with the `or` operator.

```C#
ProcessUser(id: Int) 
{
    // Guard-exit binding: 'user' and 'session' remain in scope for the rest of the method
    where user := GetUser(id) and session := GetSession(user.Id) and user.IsActive else return;

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
* MUST mutate the left-hand target with the unwrapped or validated payload and continue evaluating subsequent `and` clauses if the evaluation yields `Option::Some`.
* MUST skip the assignment, short-circuit any remaining `and` clauses, and transfer execution to the `else` branch if the evaluation yields `Option::None`.
* MUST NOT apply the `not` operator to a conditional assignment or combine a conditional assignment with the `or` operator.

```C#
SomeData 
{
    Value: Int;
    SafeAge: Int^Positive;

    UpdateFrom(ref @, rawAge: Int) 
    {
        where @.Value = rawAge as? Int^Positive and @.Value < 120 
        {
            Console.WriteLine("Value successfully updated!");
        }

        where @.SafeAge =^ rawAge - 1 
        {
            Console.WriteLine("SafeAge validated and updated!");
        }
    }
}
```

### Flow-Sensitive Type Narrowing

* MUST implicitly narrow the static type of a local variable, parameter, receiver (`@`), or immutable field path when evaluated with the `is` or `is not` operator against a variant subtype (`.Variant` or `Data::Variant`), a type guard (`^Guard` or `Type^Guard`), or a weak interface member (`.Member`).
* MUST apply positive type narrowing (`expr is Target`) to all subsequent clauses to the right within the same `and` condition chain and throughout the positive branch of the `where` statement.
* MUST apply negative type narrowing (`expr is not Target`) to the `else` branch of a `where` statement, refining `expr` to `Target` within the `else` scope.
* MUST apply type narrowing to all subsequent statements in the enclosing lexical scope following a guard-exit statement (`where expr is Target else return;`), a stage filter (`where expr is Target;`), or a negated early-exit branch (`where expr is not Target => return;`).
* MUST narrow an expression of a tagged `Data` type to the single remaining variant subtype (`Data::Variant`) in the `else` branch or following an unconditional early-exit branch if all other declared variants of that `Data` type are excluded by `is` or `is not` checks.
* MUST immediately invalidate flow-sensitive type narrowing on a mutable variable or field at the point where it is reassigned, switched via `=::`, or passed as a `ref` argument to a method that may mutate its active variant or violate its type guard invariant.

```C#
ProcessOrder(order: Order, rawAge: Int) 
{
    // Guard-exit narrowing: 'rawAge' is narrowed to 'Int^Positive' below
    where rawAge is Int^Positive else return;
    ProcessAge(rawAge);

    // Guard-exit narrowing on variant: 'order' is narrowed to 'Order::Draft' below
    where order is .Draft else return;
    Console.WriteLine(order.Items.Count);
}
```

## Staged Iteration

* MAY declare a staged iteration pipeline over a collection or range using `item in iterable { ... }` without a leading loop keyword.
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
    where enemy.Hp > 0;
    enemy.Hp -= 10;
}

// Unbounded conditional loop using 'in { ... }'
in 
{
    where x < 100 else break;
    x += 10;
}
```

### Filter Stages and Flattened Inner Stages

* MAY place one or more filter gates (`where Condition;`) at any point inside an `in` pipeline block.
* MUST immediately skip all subsequent statements in the `in` block for the current element and advance the innermost active stage if a `where Condition;` gate evaluates to `false` or `Option::None`.
* MUST bring variables bound via `:=` and type narrowings proven via `is` inside a `where Condition;` gate into scope for all subsequent statements below that gate within the `in` block.
* MAY place one or more stage expansion statements (`inner in iterable;`) inside an `in` block to flatten nested collections without nested blocks.
* MUST iterate over the expanded `iterable` for each upstream element that reaches the `inner in iterable;` stage, bringing `inner` into scope for all subsequent statements in the `in` block.

```C#
dept in company.Departments 
{
    where not dept.IsArchived;

    // Flattens department employees into the current pipeline without nesting
    emp in dept.Employees;
    where emp.Role is .Engineer;

    // Flattens employee tasks into the current pipeline
    task in emp.Tasks;
    where task.IsOverdue;

    Console.WriteLine("\(dept.Name): \(emp.Name) -> \(task.Title)");
}
```

### Output Stages and Multi-Sink Aggregation

* MAY declare one or more named output sinks (`out [Reducer] local_id ...`) inside a statement-level `in` pipeline block.
* MUST treat `count`, `sum`, `min`, `max`, `all`, `any`, `first`, and `last` as contextual reducer modifiers when placed immediately after the `out` keyword.
* MUST hoist and introduce every variable declared via a named `out` statement into both:
  * the remainder of the current `in` block immediately following the `out` statement, and
  * the lexical scope immediately enclosing the `in` pipeline statement after the `in` block completes.
* MAY read the current accumulated value of a named `out` binding in any subsequent statement or `where` gate positioned below that `out` statement within the same `in` block.
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
* MAY attach a trailing `where Condition` filter to any `out` statement to conditionally gate that specific output sink without halting subsequent statements in the `in` block.
* MUST automatically short-circuit and terminate the `in` pipeline early if the `in` block contains no side-effecting statements and all declared `out` sinks have reached a terminal state (`out first` has captured a value, `out all` has evaluated to `false`, or `out any` has evaluated to `true`).
* MAY evaluate an `in` pipeline as an expression using anonymous `out [Reducer] [Expression] [where Condition];` statements:
  * If no reducer is specified and the expected type is a collection, the pipeline MUST allocate the target collection, append or insert each emitted `out` value, and evaluate to that collection.
  * If a reducer (`count`, `sum`, `min`, `max`, `all`, `any`, `first`, `last`) is specified, the pipeline MUST evaluate to the reduced scalar or `Option` result.
  * If no reducer is specified and the expected type is `T?` (`Option<T>`), the pipeline MUST wrap the first emitted `out` value into `Option::Some`, immediately terminate the pipeline, and return it (or return `Option::None` if no element reaches `out`).
* MUST NOT mix anonymous `out` statements and named `out` statements within the same `in` pipeline block.

```C#
// 1. Multi-sink aggregation in a single pass with automatic type inference
order in orders by Parallel 
{
    where order is .Paid and order.Amount > 0.0f;

    net := order.Amount * (1.0f - order.Discount);

    out count paidCount;
    out sum   totalRevenue := net;
    out max   peakOrder    := net;
    out all   allVerified  := order.IsVerified;
    out any   hasFraud     := order.Risk > 90;
    out vipOrders: [+ Int] = order.Id where net > 1000.0f;
}

Console.WriteLine("Count: \(paidCount), Total: \(totalRevenue), Verified: \(allVerified)");

// 2. Reading an 'out' variable live inside the same pipeline (stateful accumulation)
item in cargoItems 
{
    out sum currentWeight := item.Weight;

    // Reads 'currentWeight' immediately: stops loading as soon as weight limit is exceeded
    where currentWeight <= 50.0f else break;

    out loadedItems := item;
}

// 3. Expression pipeline producing a collection
activeEmails: [+ String] = u in users 
{
    where u.IsActive;
    where profile := u.Profile;
    out profile.Email;
};

// 4. Expression pipeline with a reducer
totalPayroll := emp in employees 
{
    where emp.IsActive;
    out sum emp.Salary;
};
```

## Pattern Matching

* MAY omit the subject expression in a `match` construct (`match { ... }`) to evaluate a sequential table of boolean `Condition` arms.
* MUST evaluate boolean arms in a subjectless `match { ... }` sequentially from top to bottom and execute the first arm whose condition evaluates to `true`, or the `_` fallback arm.
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
