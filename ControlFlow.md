# Control Flow

Coggy strictly minimizes control flow constructs. Parentheses around conditions are forbidden. Branching and looping are exclusively handled by `if`, `match`, and `for`.

## Conditionals

Conditionals evaluate boolean expressions without enclosing parentheses.

```EBNF
IfStatement ::= "if" Expression (Block | "=>" Statement) ["else" ("if" Expression (Block | "=>" Statement) | Block | "=>" Statement)]
Block       ::= "{" Statement* "}"
```

* **Boolean Strictness**: The condition MUST evaluate to a strict `bool`. Implicit truthy/falsy casts (e.g., checking integers or pointers directly) are forbidden.
* **Expression Body**: An `if` or `else` branch MAY use the `=>` syntax followed by a single statement or expression, instead of a block `{}`. This provides a more concise syntax for simple conditionals.

```C#
// Block-bodied
if score > 90 
{
    Console.WriteLine("A");
} 
else if score > 80 
{
    Console.WriteLine("B");
} 
else 
{
    Console.WriteLine("F");
}

// Expression-bodied
if score > 90 => Console.WriteLine("A");
else if score > 80 => Console.WriteLine("B");
else => Console.WriteLine("F");
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
* **Custom Iteration (`by`)**: The optional `by` modifier delegates the traversal to a specific execution strategy or custom iterator. The identifier provided MUST resolve to a type that implements the `$Iterator` contract.

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

The `match` is used both for executing code blocks and returning values.

```EBNF
MatchExpr       ::= "match" Expression "{" MatchArm+ "}"
MatchArm        ::= Pattern ("or" Pattern)* "=>"? (Block | Expression) ","?
Pattern         ::= EnumVariant | Literal | "_"
```

* **Exhaustiveness**: A `match` MUST cover all possible values of the expression. If it does not, a wildcard `_` fallback MUST be provided.
* **Refinement**: Within a specific match arm, the matched data is implicitly refined to that type/variant, granting direct access to its payload without manual casting.
* **No Fallthrough**: Execution strictly terminates after the first matching arm. Fallthrough is physically impossible.
* **Expression Mode**: If used as an expression, every arm MUST evaluate to the same return type.

```C#
match color 
{
    .Red || .Blue
    {
        Console.WriteLine("Wow!");
    }
    .Green
    {
        Console.WriteLine("Go!");
    }
    _
    {
        Console.WriteLine("Boring color...");
    }
}

area := match shape 
{
    .Circle => Math.Pi * shape.Radius * shape.Radius,
    .Rectangle => (shape.Width * shape.Height) as float,
    _ => 0.0
};
```
