# Control Flow

## Gates and Conditionals

* MUST evaluate a `Condition` expression to `Bool`.
* MUST interpret an unparenthesized `{` token in the header of a `may`, `in`, or `match` construct as the opening delimiter of the construct's body block.
* MUST enclose any structural initializer (`Type { ... }` or `.Variant { ... }`), collection initializer (`[T] { ... }`, `Type { ... }`, or `{ ... }`), copy-update expression (`expr with { ... }`), structural variant pattern (`expr is .Variant { ... }`), or anonymous function (`params => body`) in parentheses `(...)` when evaluated inside a `where`, `must`, `must!`, `may`, `in`, `match`, `spawn by`, or `rollback by` header.
* MUST enclose any right-hand expression containing `and`, `or`, `??`, or `then ... :` in parentheses `(...)` when used inside a conditional binding (`:=`) or conditional assignment (`=` or `=^`).

### Guard Gates

* MAY declare a standalone guard gate (`must Condition;` or `must { Condition; ... }`) inside a method, anonymous function, or `spawn` block whose return type is omitted (void), `Bool`, or an Option type (`T?`).
* MUST immediately exit the enclosing method, anonymous function, or `spawn` block if any condition in a standalone `must` gate evaluates to `false` or `Option::None` (or a disposed `T weak` with `strongCount == 0`):
  * By returning void (`return;`) when the enclosing callable omits a return type.
  * By returning `false` (`return false;`) when the enclosing callable's return type is `Bool`.
  * By returning `Option::None` (`return .None;`) when the enclosing callable's return type is `T?` (`Option<T>`).
* MUST reject a standalone `must` gate lacking an `or` fallback clause at compile time if the enclosing callable declares a return type other than `Bool` or `T?`, or if the gate is placed directly inside a `concurrent` block outside of a `spawn` block.
* MAY attach an `or` fallback clause to a `must` gate (`must Condition or Fallback;` or `must { Condition; ... } or Fallback;`) in any method, anonymous function, `spawn` block, or `in` pipeline block.
* MUST require the `or` fallback clause of a `must` gate to unconditionally exit the enclosing scope via `return`, `throw`, `break`, or `continue`.
* MUST evaluate multiple conditions inside a `must { ... }` block sequentially from top to bottom and short-circuit immediately upon the first condition that evaluates to `false` or `Option::None` (or a disposed `T weak` with `strongCount == 0`).
* MUST bring all local identifiers bound via `:=` and all type narrowings proven via `is` within a `must` gate into the enclosing lexical scope for all subsequent statements below the gate.

```scatty
// 1. Standalone 'must' gate in a void method (returns void on failure)
SendNotification(userId: Int) 
{
    must 
    {
        user := FindUser(userId);
        user.IsActive;
    }

    Notify(user);
}

// 2. Standalone 'must' gate in a 'Bool' method (returns 'false' on failure)
CanPurchase(rawUser: User?, price: Float) -> Bool 
{
    must 
    {
        user := rawUser;
        user.IsVerified;
        user.Balance >= price;
    }

    return true;
}

// 3. Standalone 'must' gate in an 'Option<T>' ('T?') method (returns '.None' on failure)
FindActiveAdmin(userId: Int) -> User? 
{
    must 
    {
        user := FindUser(userId);
        user.IsActive;
        user.Role is .Admin;
    }

    return .Some(user);
}

// 4. 'must ... or' gate in a method returning 'Float', and 'where' filter inside 'in'
ProcessBatch(orders: [Order], userId: Int) -> Float 
{
    must 
    {
        orders.Count > 0;
        user := FindUser(userId);
        user.IsActive;
    } 
    or return 0.0f;

    order in orders -> total := 0.0f
    {
        where 
        {
            order is .Paid;
            order.Amount > 0.0f;
            order.OwnerId == user.Id;
        }

        total += order.Amount;
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

```scatty
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

```scatty
must x > 0;
must x > 100;

may x == 101 => Console.WriteLine("101!");
or Console.WriteLine("Not 101");

item in items -> total := 0.0f
{
    where item.IsValid;
    may item.IsRare => LogRareItem(item);
    must! item.Price >= 0.0f;

    total += item.Price;
}
```

### Conditional Binding

* MUST evaluate the right-hand expression of a conditional binding (`:=`) to an Option type (`T?`) or a weak reference type (`T weak`).
* MUST bind the unwrapped payload (or promoted strong reference `T`) to the new local identifier and continue evaluating the condition if the expression evaluates to `Option::Some` (or a live `T weak` with `strongCount > 0`).
* MUST short-circuit the condition if the expression evaluates to `Option::None` (or a disposed `T weak` with `strongCount == 0`).
* MAY chain multiple conditional bindings, conditional assignments, and boolean expressions using the `and` operator or across sequential lines inside a `where { ... }`, `must { ... }`, or `must! { ... }` block.
* MUST bring any local identifier introduced by a conditional binding (`:=`) into scope for all subsequent clauses to its right within an `and` chain, for all subsequent lines within a `where { ... }`, `must { ... }`, or `must! { ... }` block, and:
  * within the positive branch of a `may` statement,
  * within the matching arm body of a subjectless `match { ... }` construct, or
  * within the enclosing lexical scope for all statements following a `where`, `must`, or `must!` gate.
* MUST NOT access a local identifier introduced by a conditional binding inside an `or` fallback branch.
* MUST NOT declare duplicate local identifiers across multiple conditional bindings within the same condition or `where` / `must` / `must!` block.
* MUST NOT apply the `not` operator to a conditional binding or combine a conditional binding with the boolean `or` operator.

### Conditional Assignment

* MUST evaluate the right-hand expression to an Option type (`T?`) when using the standard assignment operator (`=`) within a condition clause.
* MUST resolve the left-hand `AssignablePath` to a mutable target of the unwrapped payload type `T`.
* MUST NOT use the `=` assignment operator with non-Option types within a condition clause.
* MAY use the guarded assignment operator (`=^`) within a condition clause when the left-hand `AssignablePath` resolves to a mutable target of a guarded type (`T^Guard`) and the right-hand expression evaluates to the underlying base type `T`.
* MUST evaluate `AssignablePath =^ Expression` within a condition clause as `AssignablePath = (Expression) as? T^Guard`.
* MAY chain conditional assignments with boolean expressions and conditional bindings using `and` or inside a `where { ... }`, `must { ... }`, or `must! { ... }` block.
* MUST mutate the left-hand target with the unwrapped or validated payload and continue evaluating subsequent clauses if the evaluation yields `Option::Some`.
* MUST skip the assignment and short-circuit remaining clauses if the evaluation yields `Option::None`.
* MUST NOT apply the `not` operator to a conditional assignment or combine a conditional assignment with the boolean `or` operator.

```scatty
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
* MUST bring any local identifier (`local_id`) bound within an `expr is Pattern` clause into scope for all subsequent clauses to its right within an `and` chain, for all subsequent lines within a `where { ... }`, `must { ... }`, or `must! { ... }` block, throughout the positive branch of a `may` statement, and for all subsequent statements in the enclosing lexical scope following a `where`, `must`, or `must!` gate.
* MUST NOT declare local variable bindings inside an `is not` expression or combine a binding `is` expression with the boolean `or` operator.
* MUST apply positive type narrowing (`expr is Target`) to all subsequent clauses within the same `and` chain or `where { ... }` / `must { ... }` / `must! { ... }` block, throughout the positive branch of a `may` statement, and to all statements in the enclosing scope following a `where`, `must`, or `must!` gate.
* MUST apply negative type narrowing (`expr is not Target`) to the `or` branch of a `may` statement, refining `expr` to `Target` within the `or` scope.
* MUST apply type narrowing to all subsequent statements in the enclosing lexical scope whenever a `may` branch testing a negated condition (`may expr is not Target => return;`) unconditionally terminates execution of the current scope via `return`, `throw`, `break`, or `continue`.
* MUST narrow an expression of a tagged `Data` type to the single remaining variant subtype (`Data::Variant`) in the `or` branch or following an unconditional early-exit branch or `where` / `must` / `must!` gate if all other declared variants of that `Data` type are excluded.
* MUST immediately invalidate flow-sensitive type narrowing on a mutable variable or field at the point where it is reassigned, switched via `=::`, or passed as a `ref` argument to a method that may mutate its active variant or violate its type guard invariant.
* MUST invalidate flow-sensitive narrowing to a forwarded variant subtype (`Parent::Variant`) whenever any embedded field along the variant-forwarding chain is reassigned, switched via `=::`, or passed as a `ref` argument to a method that can mutate its active variant.

## Staged Iteration

* MAY declare a staged iteration pipeline over a collection or range using `item in iterable { ... }` or `item in iterable => Statement;` without a leading loop keyword.
* MUST treat all non-`ref` stage bindings (`item in iterable` and destructured bindings `(a, b) in iterable`) as immutable read-only views for the duration of each iteration.
* MUST NOT reassign a non-`ref` stage binding, mutate its fields, switch its active variant via `=::`, pass it or its fields as `ref` arguments, or invoke mutating (`ref @`) methods on it.
* MAY prefix a single-identifier stage binding with `ref` (`ref item in collection { ... }`) to declare a mutable in-place slot binding over `collection`.
* MUST unpack tuple elements in the stage header using a destructuring pattern (`(a, b) in iterable { ... }`), requiring the binding count to match the tuple element count and permitting `_` to discard elements.
* MUST evaluate the upper bound of a range expression (`start..end` or `start..end..step`) as exclusive.
* MAY delegate the execution schedule of a non-`ref` pipeline using the `by` modifier (`item in iterable by Schedule { ... }`), resolving the identifier to a type implementing the `~Iterator` interface.
* MUST NOT combine a `ref` stage binding with the `by` schedule modifier.
* MAY omit both the stage binding and the iterable (`in { ... }`) to declare an unbounded loop that executes repeatedly until terminated via `break`, `return`, or `throw`.
* MAY use `break` inside an `in` block to immediately terminate the entire pipeline.
* MAY use `continue` inside an `in` block to immediately skip the remainder of the current iteration and advance the innermost active stage.
* MUST NOT use `break` or `continue` outside the lexical body of an `in` block or across a lambda or `spawn` boundary.

```scatty
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

// In-place mutation of collection elements with 'where' filter
ref enemy in enemies 
{
    where enemy.Hp > 0;
    enemy.Hp -= 10;
}

// Unbounded conditional loop using 'in { ... }'
in 
{
    must x < 100 or break;
    x += 10;
}
```

### Iteration Protocol Resolution

* MUST resolve a stage source lacking a `by` modifier according to the following descending priority order:
  1. Built-in range expressions (`start..end` or `start..end..step`) for non-`ref` iteration only.
  2. Built-in collections (`[T]`, `[T; N]`, `Array<T>`, `[T..]`, `[ref T..]`, `String`, `RuneString`, `[K -> V]`, `[T] soa`, `[T; N] soa`, `Array<T> soa`, `[$: T]`, and `[K: V]`).
  3. Any `Data` type (or constrained generic parameter) implementing `~CursorIndexable<TCursor, TValue>`, traversing cursors via `FirstCursor` and `NextCursor` and reading elements via `GetAt`.
  4. Any `Data` type (or constrained generic parameter) that does not implement `~CursorIndexable` and implements `~Indexable<Int, TValue>` together with the weak contract `Count: Int`, evaluating `Count` once prior to loop entry and traversing integer indices `0..Count` via `Get`.
* MUST reject an `in` iteration source at compile time if it implements multiple distinct specializations of `~CursorIndexable`, or implements multiple distinct specializations of `~Indexable<Int, T>` without `~CursorIndexable`.
* MUST evaluate `Get` or `GetAt` once per iteration to bind the immutable stage variable in a non-`ref` loop over a custom `Data` type.

### Mutable `ref` Stage Bindings

* MUST require the target `collection` of a `ref` stage (`ref item in collection` or `ref inner in collection;`) to be a mutable lvalue path governed by the Law of Exclusivity.
* MUST permit `ref item in collection` exclusively on:
  * Built-in contiguous addressable collections: `[T]`, `[T; N]`, `Array<T>`, `[ref T..]`, `RuneString`, and `[K -> V]`.
  * Built-in `soa` collections: `[T] soa`, `[T; N] soa`, and `Array<T> soa`.
  * `String` exclusively within an `unsafe` block or `unsafe` method.
  * Any `Data` type implementing `~CursorIndexable<TCursor, TValue>` with the weak contract `SetAt`.
  * Any `Data` type (not implementing `~CursorIndexable`) implementing `~Indexable<Int, TValue>` with both weak contracts `Count` and `Set`.
* MUST reject `ref item in collection` at compile time on ranges, read-only slices (`[T..]`), `String` outside `unsafe`, hashed collections (`[$: T]`, `[K: V]`), custom `Data` types omitting `Set` / `SetAt`, `static` or `astatic` storage, rvalue expressions, and pipelines declaring a `by` schedule modifier.
* MUST evaluate any index expressions within `collection`'s lvalue path (such as `matrix[GetRow()]`) exactly once prior to entering the loop.
* MUST bind `item` in `ref item in collection` as an in-place slot alias corresponding to the active element:
  * On built-in contiguous addressable collections, accesses and mutations of `item` operate directly on the element's memory slot without bounds checking.
  * On built-in `soa` collections, accesses and mutations of `item` operate identically to indexed `soa` access (`collection[_i]`): field accesses and leaf-field `ref` borrows target the parallel arrays directly, whole-element assignment performs Scatter, whole-element reads perform Gather, and passing the whole element by `ref` (`ref item`) or invoking a `ref @` method directly on `item` MUST be rejected at compile time.
  * On custom `~CursorIndexable<TCursor, TValue>` types with `SetAt` and custom `~Indexable<Int, TValue>` types with `Count` and `Set`, reads of `item` invoke `GetAt` / `Get`, and mutations of `item` unpack into the read-modify-write sequences defined in the Indexing Operator specification, propagating `throws` according to the concrete `Get` / `Set` / `GetAt` / `SetAt` signatures.
* MUST NOT bind a mutating instance method (`item.MutatingMethod`) directly as a stack closure (`fn`) when `item` aliases a whole `soa` element or an element of a custom `~Indexable` or `~CursorIndexable` type.
* MUST enforce standard `TypeGuards` rules on `item` when the element type of `collection` is guarded (`[T^Guard]`), requiring guarded assignment operators (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`) and element-level post-`ref` / post-`ref @` invariant checks.
* MUST evaluate all enclosing `^Guard` invariants along `collection`'s path (from innermost to outermost) once upon exiting a `ref` stage (on normal completion, `break`, `return`, or error propagation) when `collection` itself or an enclosing container along its path is governed by a type guard, obeying active build-mode validation rules or checking in both Debug and Release modes when postfix `strict;` is applied to the block-bodied `in` statement.

```scatty
// Range iteration (0 through 9); 'i' is immutable
i in 0..10 
{
    Console.WriteLine(i);
}

// Reverse range with step (10 down to 1)
i in 10..0..-1 
{
    Console.WriteLine(i);
}

// Non-'ref' iteration: 'enemy' is an immutable read-only view
enemy in enemies 
{
    Console.WriteLine(enemy.Hp);
    enemy.Hp -= 10; // ERROR: 'enemy' is immutable
}

// Mutable slot iteration on built-in collections (including 'soa')
ref enemy in enemies 
{
    where enemy.Hp > 0;
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

* MAY place one or more filter gates (`where Condition;` or `where { Condition; ... }`) at any point inside an `in` pipeline block.
* MUST NOT use a `where` statement outside the lexical body of an `in` pipeline block or across a nested anonymous function or `spawn` boundary.
* MUST evaluate multiple conditions inside a `where { ... }` block sequentially from top to bottom and short-circuit immediately upon the first condition that evaluates to `false` or `Option::None` (or a disposed `T weak` with `strongCount == 0`).
* MUST immediately skip all subsequent statements in the `in` block for the current element and advance the innermost active stage if a `where` gate evaluates to `false` or `Option::None` (or a disposed `T weak` with `strongCount == 0`).
* MUST bring variables bound via `:=` and type narrowings proven via `is` inside a `where` gate into scope for all subsequent statements below that gate within the `in` block.
* MAY place one or more stage expansion statements (`inner in iterable;`) inside an `in` block to flatten nested collections without nested blocks.
* MUST iterate over the expanded `iterable` for each upstream element that reaches the `inner in iterable;` stage, bringing `inner` into scope for all subsequent statements in the `in` block.
* MUST permit a mutable flattened inner stage (`ref inner in iterable;`) if and only if `iterable` is a mutable lvalue path rooted in a mutable outer variable or an enclosing `ref` stage binding, rejecting `ref inner in outer.Items;` when `outer` is an immutable non-`ref` stage binding.

```scatty
dept in company.Departments 
{
    where not dept.IsArchived;

    emp in dept.Employees;
    where emp.Role is .Engineer;

    task in emp.Tasks;
    where task.IsOverdue;

    Console.WriteLine("\(dept.Name): \(emp.Name) -> \(task.Title)");
}

ref dept in company.Departments 
{
    ref emp in dept.Employees;
    where emp.Role is .Engineer;

    emp.Bonus += 500;
}
```

### Output Bindings

* MAY declare one or more comma-separated output bindings in the `in` header following the `->` token (`-> local_id := Expression` or `-> local_id: Type = Expression`).
* MUST place the `->` output binding list after the `iterator` and optional `by` schedule clause, and immediately prior to the pipeline body (`{ ... }` or `=> Statement;`).
* MUST NOT enclose the `->` output binding list in parentheses.
* MUST evaluate each output binding's initialization expression once prior to executing the first iteration of the pipeline.
* MUST NOT declare duplicate `local_id` identifiers within the same `->` output binding list.
* MAY read, mutate, and reassign output bindings within the body of the `in` pipeline using standard statements and operators.
* MAY annotate an output binding with a guarded type (`local_id: Type^Guard = Expression` or `local_id: Type^Guard =^ Expression`).
* MUST treat a guarded output binding as its unguarded base type `Type` during initialization and throughout the execution of the `in` body, and evaluate the governing `^Guard` invariant once upon completion of the `in` pipeline according to the active build-mode validation rules (or in both Debug and Release modes if the `strict` modifier is applied to the `in` statement).

#### Statement-Level Evaluation

* MUST hoist all output bindings declared in the `->` clause of a statement-level `in` pipeline into the immediately enclosing lexical scope upon completion of the pipeline.
* MUST NOT shadow an identifier already declared in the enclosing lexical scope when declaring an output binding on a statement-level `in` pipeline.
* MUST require every hoisted output binding of a statement-level `in` pipeline to be read at least once in the enclosing lexical scope following the pipeline; an unreferenced hoisted output binding MUST be rejected at compile time.

#### Expression-Level Evaluation

* MUST declare at least one output binding via `->` when an `in` pipeline is evaluated as an expression.
* MUST scope output bindings declared in an expression-level `in` pipeline to the pipeline body. They MUST NOT be hoisted into the enclosing lexical scope.
* MUST evaluate an expression-level `in` pipeline declaring a single output binding (`-> result := init`) directly to the final value and static type of that binding upon termination of the pipeline.
* MUST evaluate an expression-level `in` pipeline declaring two or more output bindings (`-> out1 := init1, out2 := init2`) to a tuple containing the final values of the output bindings in their declaration order upon termination of the pipeline.
* MUST declare a `{ ... }` block body on an expression-level `in` pipeline; an expression-level `in` pipeline MUST NOT use the `=> Statement;` body form.
* MUST declare a `{ ... }` block body when applying the `strict` modifier to an `in` pipeline (`... { ... } strict;`), and MUST reject `strict;` on an `in` pipeline that has neither guarded output bindings nor a `ref` stage over a guarded collection or guarded container path.

```scatty
// 1. Statement-level pipeline hoisting multiple output bindings into the enclosing scope
order in orders -> paidCount := 0, totalRevenue := 0.0f, vipOrders := [Int].New()
{
    where order is .Paid and order.Amount > 0.0f;

    net := order.Amount * (1.0f - order.Discount);
    paidCount += 1;
    totalRevenue += net;

    may net > 1000.0f => vipOrders.Add(order.Id);
}

Console.WriteLine("Count: \(paidCount), Revenue: \(totalRevenue), VIPs: \(vipOrders.Count)");

// 2. Early termination reading an output binding mid-pipeline
item in cargoItems -> currentWeight := 0.0f, loadedItems: [Item] = ({})
{
    must currentWeight + item.Weight <= 50.0f or break;

    currentWeight += item.Weight;
    loadedItems.Add(item);
}

Console.WriteLine("Loaded \(loadedItems.Count) items (\(currentWeight) kg)");

// 3. Expression-level pipeline with a single output binding (evaluates to 'Float')
netTotal := order in orders -> sum := 0.0f 
{
    where order is .Paid;
    sum += order.Amount;
};

// 4. Expression-level pipeline with multiple output bindings (evaluates to '(Int, Int)')
(minScore, maxScore) := s in scores -> minVal := Int.MaxValue, maxVal := Int.MinValue 
{
    may s < minVal => minVal = s;
    may s > maxVal => maxVal = s;
};
```

## Pattern Matching

* MAY omit the subject expression in a `match` construct (`match { ... }`) to evaluate a sequential table of boolean `Condition` arms.
* MUST evaluate boolean arms in a subjectless `match { ... }` sequentially from top to bottom and execute the first arm whose condition evaluates to `true`, or the `_` fallback arm.
* MUST exhaustively cover all possible variants of the evaluated expression, or provide a wildcard `_` fallback.
* MAY match the subject expression against variant patterns (`.Variant`), structural patterns (`{ Field: sub_pattern }`), tuple patterns (`(sub_pattern, sub_pattern)`), type guard patterns (`^Guard`), literals, or the wildcard `_`.
* MAY nest patterns recursively within variant payload patterns, structural field patterns, and tuple patterns.
* MAY bind any matched variant payload field, structural field, or tuple element to a local variable (`local_id`) or discard it using `_`.
* MAY mix named (`FieldName: sub_pattern`) and positional (`sub_pattern`) pattern arguments within a variant pattern (`.Variant(...)`).
* MUST bind a named pattern argument (`FieldName: sub_pattern`) explicitly to the variant payload field with the specified identifier.
* MUST bind a positional pattern argument (`sub_pattern`) to the first declared variant payload field not yet matched within the variant pattern, dynamically shifting the expected sequence when a named pattern argument is evaluated.
* MUST require a variant pattern containing only positional pattern arguments to match the exact payload field count of the variant, using `_` to discard unused fields.
* MAY omit remaining unmatched payload fields in a variant pattern when at least one named pattern argument is specified.
* MAY match accessible structural or shared fields of a `Data` type - including unambiguous accessible forwarded structural fields of embedded fields - by field identifier using `{ FieldName: sub_pattern }`, either standalone or following a variant pattern (`.Variant(...) { FieldName: sub_pattern }`), and omit remaining fields; matching an ambiguous forwarded field MUST be rejected at compile time.
* MAY specify multiple comma-separated patterns (`pattern1, pattern2`) on a single match arm.
* MUST require every pattern in a comma-separated pattern list (`pattern1, pattern2`) to bind the exact same set of `local_id` identifiers with identical static types if any local variable bindings are declared.
* MUST implicitly refine the matched expression and any matched fields to the evaluated variant subtype or type guard within the match arm.
* MUST evaluate every match arm to the exact same return type if the `match` construct is utilized as an expression.
* MUST resolve variant patterns (`.Variant`) applied to a `Data` type possessing an accessible embedded variant field against its forwarded variants and refine the matched subject to `Parent::Variant` within the matching arm.

```scatty
match response 
{
    // Positional variant pattern with literal and '_' discard
    .Error(404, msg, _) => HandleNotFound(msg);

    // Mixed named and positional variant pattern:
    // 'Message' binds by name; positional '^ServerError' binds to first unmatched field ('Code');
    // remaining payload fields are omitted
    .Error(Message: msg, ^ServerError) => HandleServerError(response.Code, msg);

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
* MUST evaluate the type guard's `where` invariant against the target value at runtime and match if the invariant evaluates to `true`.
* MUST implicitly refine the static type of the matched expression or field to `BaseType^Guard` within the body of a matching arm.
* MAY attach an `and Condition` clause to a match arm following its pattern list.
* MUST evaluate the `and` condition to `Bool`.
* MUST bring any variables bound by the arm's pattern into scope within the `and` condition.
* MUST bind `@` within a match arm's `and` condition to the evaluated match subject.
* MUST bypass the match arm and continue evaluating subsequent arms if the pattern matches but its `and` condition evaluates to `false`.
* MUST NOT treat a match arm governed by a `^Guard` pattern, a nested literal or structural field pattern, or an `and` condition as exhaustive coverage for the underlying type or variant; an unguarded fallback arm or `_` wildcard MUST be provided in an exhaustive `match`.

```scatty
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
* MUST NOT use the `fall` statement to transfer control into a match arm that declares local variable bindings in its pattern or `and` condition.
* MUST NOT use the `fall` statement to transfer control into a match arm whose body relies on implicit type narrowing of the matched subject expression or its fields (such as accessing variant-specific payload fields, invoking constrained-receiver methods, or passing the subject where a narrowed `Data::Variant` or `BaseType^Guard` type is required) unless the source arm statically guarantees the exact same narrowing.
* MUST NOT use the `fall` statement within the final match arm.

```scatty
Color 
{
    ::Red;
    ::Green(X: Int, Y: Int);
}

c: Color = Color::Red;

match c 
{
    .Red 
    {
        Console.WriteLine("Red encountered, falling through to shared logic");
        fall; // Valid: target arm does not rely on 'c' being narrowed to 'Color::Green'
    }
    .Green 
    {
        Console.WriteLine("Handled Red or Green");
    }
}

match c 
{
    .Red 
    {
        fall; // ERROR: target arm relies on 'c' being narrowed to 'Color::Green'
    }
    .Green 
    {
        Console.WriteLine(c.X);
    }
}
```

### Non-Exhaustive Pattern Matching

* MUST NOT require exhaustive variant coverage or a wildcard `_` fallback when evaluating `match!` or `match?`.
* MUST immediately trigger runtime Abort or Fatal Panic mechanics if a `match!` construct fails to match the evaluated expression against any provided arm.
* MUST evaluate a `match?` construct used as an expression to `T?` (`Option<T>`), regardless of the number of match arms.
* MUST implicitly wrap the evaluated value of a matching `match?` arm into `Option::Some`.
* MUST yield `Option::None` if a `match?` expression fails to match any provided arm (or perform a no-op if `match?` is used as a void statement).

```scatty
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
