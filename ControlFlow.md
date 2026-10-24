# Control Flow

## Conditionals

* MUST evaluate a `Condition` expression to `Bool`.

### Conditional Binding

* MUST evaluate the right-hand expression to an Option type (`T?`).
* MUST bind the unwrapped payload to the new local identifier and execute the block if the Option evaluates to `Some`.
* MUST skip execution of the block if the Option evaluates to `None`.

```C#
maybeUser: User? = GetUser();

if user := maybeUser 
{
    Console.WriteLine(user.Name); // 'user' is a new local variable
}
```

### Conditional Assignment

* MUST evaluate the right-hand expression to an Option type (`T?`) when using the standard assignment operator (`=`).
* MUST resolve the left-hand `AssignablePath` to a mutable target of the unwrapped payload type `T`.
* MUST NOT use the `=` assignment operator with non-Option types within a condition.
* MAY use the guarded assignment operator (`=^`) within a condition when the left-hand `AssignablePath` resolves to a mutable target of a guarded type (`T^Guard`) and the right-hand expression evaluates to the underlying base type `T`.
* MUST evaluate `AssignablePath =^ Expression` within a condition as `AssignablePath = (Expression) as? T^Guard`.
* MUST mutate the left-hand target with the unwrapped or validated payload and execute the block if the evaluation yields `Some`.
* MUST skip the assignment and transfer execution to the `else` branch if the evaluation yields `None`.

```C#
SomeData 
{
    Value: Int;
    SafeAge: Int^Positive;

    UpdateFrom(ref @, rawAge: Int) 
    {
        // Unwraps the Option from 'as?' directly into the field '@.Value'
        // If rawAge is <= 0, the Option is None, and @.Value is NOT modified
        if @.Value = rawAge as? Int^Positive 
        {
            Console.WriteLine("Value successfully updated!");
        }

        // Guarded conditional assignment: deduces 'Int^Positive' from '@.SafeAge'
        // Evaluates as: if @.SafeAge = (rawAge - 1) as? Int^Positive
        if @.SafeAge =^ rawAge - 1 
        {
            Console.WriteLine("SafeAge validated and updated!");
        }
    }
}
```

## Loops

* MUST evaluate a single `Condition` expression as a boolean to perform conditional looping.
* MUST bind the iteration element to an local variable when using the `in` operator.
* MUST evaluate the upper bound of a range expression (`..`) as exclusive, omitting the right-hand boundary value from the iteration sequence.
* MAY delegate the traversal strategy using the `by` modifier.
* MUST resolve the identifier provided to the `by` modifier to a type implementing the `~Iterator` interface.

```C#
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
* MUST throw a runtime exception if a `match!` construct fails to match the evaluated expression against any provided arm.
* MUST evaluate a `match?` construct containing exactly one match arm to an `Option` type.
* MUST yield `Option::None` if a single-arm `match?` fails to match.
* MUST evaluate a `match?` construct containing two or more match arms to a `Result` type.
* MUST yield `Result::Error` if a multi-arm `match?` fails to match.
* MUST implicitly wrap the evaluated value of a successful `match?` arm into the corresponding `Option` or `Result` success variant.

```C#
// match! throws an exception if the 'shape' is not a 'Circle'
match! shape.Type  
{
    .Circle(r) => Console.WriteLine("Wow!");
}

// 'area' becomes 'Result::Error' if the 'shape' is neither 'Circle' or 'Rectangle'
area := match? shape.Type 
{
    .Circle(r) => Math.Pi * r * r;
    .Rectangle(w, h) => (w * h) as Float;
};

// 'circleArea' becomes 'Option::None' if the 'shape' is not a 'Circle'
circleArea := match? shape.Type 
{
    .Circle(r) => Math.Pi * r * r;
};
```
