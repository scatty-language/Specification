# Control Flow

## Conditionals

```EBNF
IfStatement           ::= "if" Condition (Block | "=>" Statement) ["else" IfTail]
IfTail                ::= "if" Condition (Block | "=>" Statement) | Block | "=>" Statement
Condition             ::= Expression | ConditionalBinding | ConditionalAssignment

ConditionalBinding    ::= LocalId ":=" Expression
ConditionalAssignment ::= AssignablePath "=" Expression
AssignablePath        ::= Identifier ("." Identifier)*

Block                 ::= "{" Statement* "}"
```

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

* MUST evaluate the right-hand expression to an Option type (`T?`).
* MUST resolve the left-hand `AssignablePath` to a mutable target of the unwrapped payload type `T`.
* MUST NOT use the `=` assignment operator with non-Option types within a condition.
* MUST mutate the left-hand target with the unwrapped payload and execute the block if the Option evaluates to `Some`.
* MUST skip the assignment and transfer execution to the `else` branch if the Option evaluates to `None`.

```C#
SomeData 
{
    Value: Int;

    UpdateFrom(ref @, rawAge: Int) 
    {
        // Unwraps the Option from 'as?' directly into the field '@.Value'
        // If rawAge is <= 0, the Option is None, and @.Value is NOT modified
        if @.Value = rawAge as? Int^Positive 
        {
            Console.WriteLine("Age successfully updated!");
        }
    }
}
```

## Loops

```EBNF
ForLoop         ::= "for" (Condition | Iteration) Block
Condition       ::= Expression
Iteration       ::= Identifier "in" Iterator ["by" Identifier]
Iterator        ::= Expression | Range
Range           ::= Expression ".." Expression [".." Expression]
```

* MUST evaluate a single `Condition` expression as a boolean to perform conditional looping.
* MUST bind the iteration element to an local variable when using the `in` operator.
* MAY delegate the traversal strategy using the `by` modifier.
* MUST resolve the identifier provided to the `by` modifier to a type implementing the `~Iterator` interface.

```C#
// Conditional loop
for x < 100 
{
    x += 10;
}

// Iterator loop over a range
for i in 0..10 
{
    Console.WriteLine(i);
}

// Iterator loop over a range with 2 step
for i in 0..10..+2
{
    Console.WriteLine(i);
}

// Iterator loop over a reverse range
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

```EBNF
MatchExpr       ::= "match" ["!" | "?"] Expression "{" MatchArm+ "}"
MatchArm        ::= Pattern ("or" Pattern)* "=>"? (Block | Expression) ";"?
Pattern         ::= "." Identifier ["(" LocalId ("," LocalId)* ")"] | Literal | "_"
FallStmt        ::= "fall" ";"
```

* MUST exhaustively cover all possible variants of the evaluated expression, or provide a wildcard `_` fallback.
* MAY bind the payload of a Variant Field to local identifiers using parentheses.
* MUST implicitly refine the matched expression to the evaluated variant within the match arm if payload binding is omitted.
* MUST evaluate every match arm to the exact same return type if the `match` construct is utilized as an expression.

### Execution Flow and Explicit Fallthrough

* MUST terminate execution of the `match` construct immediately after the first matching arm finishes (no implicit fallthrough).
* MAY use the `fall` statement within a block-bodied match arm to immediately transfer execution to the body of the next sequential arm without evaluating its pattern.
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
