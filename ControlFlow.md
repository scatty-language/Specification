# Control Flow

In Coggy branching and looping are exclusively handled by `if`, `else`, `match`, and `for`.

## Conditionals

Conditionals evaluate boolean expressions or unpack Option types (`T?`).

```EBNF
IfStatement           ::= "if" Condition (Block | "=>" Statement) ["else" IfTail]
IfTail                ::= "if" Condition (Block | "=>" Statement) | Block | "=>" Statement
Condition             ::= Expression | ConditionalBinding | ConditionalAssignment

ConditionalBinding    ::= LocalId ":=" Expression
ConditionalAssignment ::= AssignablePath "=" Expression
AssignablePath        ::= Identifier ("." Identifier)*

Block                 ::= "{" Statement* "}"
```

* **Strict Boolean**: If the `Condition` is an `Expression`, it MUST evaluate to `Bool`.

### Conditional Binding

Introduces a new local variable to hold the unwrapped payload.

* **Option Requirement**: The RHS `Expression` MUST evaluate to an Option type (`T?`).
* **Execution**: If the Option is `Some`, the payload is bound to the new `Identifier` (type `T`), and the block executes. If `None`, it skips.

```C#
maybeUser: User? = GetUser();

if user := maybeUser 
{
    Console.WriteLine(user.Name); // 'user' is a new local variable
}
```

### Conditional Assignment

* **Mutation of Existing State**: Unwraps an Option and assigns the payload directly to an existing variable or field.
* **Type Safety Guard**: The RHS `Expression` MUST evaluate to an Option type (`T?`). The LHS `AssignablePath` MUST be a mutable target of type `T`.
* **Non-Option Types**: Using `=` with non-Option types in an `if` condition is FORBIDDEN. This entirely eliminates the classic C/C++ assignment typo bug.
* **Execution**: If the Option is `Some`, the payload is assigned to the LHS target, and the block executes. If the Option is `None`, the assignment is skips, and the execution skips to the `else` branch.

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

Coggy unifies all iteration and conditional looping into a single `for` keyword.

```EBNF
ForLoop         ::= "for" (Condition | Iteration) Block
Condition       ::= Expression
Iteration       ::= Identifier "in" Iterator ["by" Identifier]
Iterator        ::= Expression | Range
Range           ::= Expression ".." Expression [".." Expression]
```

* **Conditional Loop**: Providing a single boolean expression acts as a traditional `while` loop.
* **Range/Iterator Loop**: The `in` operator binds the iteration element to an immutable local variable.
* **Custom Iteration (`by`)**: The optional `by` modifier delegates the traversal to a specific execution strategy or custom iterator. The identifier provided MUST resolve to a type that implements the `~Iterator` interface.

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

The `match` construct is used both for executing code blocks and returning values.

```EBNF
MatchExpr       ::= "match" Expression "{" MatchArm+ "}"
MatchArm        ::= Pattern ("or" Pattern)* "=>"? (Block | Expression) ","?
Pattern         ::= "." Identifier ["(" LocalId ("," LocalId)* ")"] | Literal | "_"
FallStmt        ::= "fall" ";"
```

* **Exhaustiveness**: A `match` MUST cover all possible variants of the expression. If it does not, a wildcard `_` fallback MUST be provided.
* **Payload Binding**: When matching against a Variant Field, the payload MAY be bound to local identifiers using parentheses (e.g., `.Hmm(x, y)`).
* **Implicit Refinement**: If payload binding is omitted, but a variant is matched, the matched expression is implicitly refined to that variant within the match arm.
* **Expression Mode**: If used as an expression, every arm MUST evaluate to the exact same return type.

### Execution Flow and Explicit Fallthrough

* **No Implicit Fallthrough**: By default, execution terminates after the first matching arm finishes.
* **Explicit `fall`**: The `fall` statement MAY be used inside a block-bodied match arm to immediately transfer execution to the body of the next sequential arm, bypassing its pattern evaluation.
* **Fallthrough Restrictions**:
  1. **Statement Only**: `fall` is FORBIDDEN if the `match` construct is being evaluated as an expression.
  2. **No Payload Destination**: `fall` MUST NOT transfer control into a match arm that binds payload variables.
  3. **Not Last**: `fall` MUST NOT be used in the last match arm.

```C#
match statusCode 
{
    404 => Console.WriteLine("Not Found"),
    
    // Cascading logic using 'fall'
    401, 403 
    {
        Console.WriteLine("Access Denied");
        fall; // Transfers control directly to the '_' block
    },
    
    _ => Console.WriteLine("Logging error...")
}

match some.VariedField 
{
    .Wow 
    {
        Console.WriteLine("Wow!");
        fall; // ERROR: Cannot fall into '.Hmm' because it binds 'x' and 'y'
    },
    
    .Hmm(x, y) 
    {
        Console.WriteLine(x + y);
    }
}

area := match shape.Type 
{
    .Circle(r) => Math.Pi * r * r,
    .Rectangle(w, h) => (w * h) as Float,
    _ => 0.0f
};
```
