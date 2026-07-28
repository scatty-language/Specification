# Tuples

Tuples provide concise syntax to group multiple values into a single, anonymous inline-allocated Data structure.

```EBNF
TupleType     ::= "(" TupleTypeElem ("," TupleTypeElem)+ ")"
TupleTypeElem ::= [PublicId ":"] Type

TupleExpr     ::= "(" TupleExprElem ("," TupleExprElem)+ ")"
TupleExprElem ::= [PublicId ":"] Expression
```

* **Minimum Size**: Tuple MUST contain at least two elements. Single-element tuples are FORBIDDEN.
* **Value Semantics**: Tuples adhere to standard Value Semantics. They are stack-allocated and copied by value upon assignment or parameter passing.
* **Implicit Naming**: If an element is declared without an explicit name, the compiler automatically assigns it a positional identifier starting from 1 (e.g., `Item1`, `Item2`, `Item3`).

```C#
GetBounds() -> (Min: Int, Max: Int) 
{
    return (Min: 0, Max: 100);
}

point: (Int, Int) = (10, 20);

Console.WriteLine(point.Item1); // Prints 10

user := (Id: 1, Name: "Coggy");
Console.WriteLine(user.Name);
```

## Type Equivalence and Assignment

Tuple types are resolved by their structural sequence of types.

* **Positional Assignment**: MAY assign one tuple to another if and only if their underlying types match exactly in positional order.

```C#
pointA: (X: Int, Y: Int) = (10, 20);
pointB: (Width: Int, Height: Int) = (5, 5);
unnamed: (Int, Int) = (0, 0);

pointA = pointB;
unnamed = pointA;

Console.WriteLine(pointA.X); // Evaluates to 5
```

## Destructuring

Tuples can be unpacked directly into multiple distinct local variables using destructuring assignment.

```EBNF
DestructureStmt ::= "(" LocalId ("," LocalId)+ ")" ("=" | ":=") Expression ";"
```

* **Declaration (`:=`)**: Introduces entirely new local variables into the current scope.
* **Mutation (`=`)**: Assigns the unpacked values to existing, mutable local variables.
* **Discard (`_`)**: The wildcard identifier `_` MAY be used in place of a `LocalId` to discard an element without binding it to a variable.

```C#
// Declaration destructuring
(min, max) := GetBounds();
Console.WriteLine(min);

// Mutation destructuring
min = -10;
max = 10;
(min, max) = GetBounds(); 

// Discarding element
(id, _) := (Id: 42, Name: "Ignored");
```
