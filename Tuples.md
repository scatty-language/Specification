# Tuples

* MUST allocate tuples as anonymous, inline contiguous `Data` structures.
* MUST contain at least two elements in a tuple declaration or instantiation.
* MUST NOT declare or instantiate a single-element tuple.
* MUST adhere to Value Semantics for all tuple copies, parameter passing, and assignments.
* MAY define explicit element names within a tuple type or literal.
* MUST automatically assign 1-based positional names (`Item1`, `Item2`, ...) to tuple elements declared without an explicit identifier.
* MUST allow member access via explicitly declared element names or positional identifiers.

```scatty
GetBounds() -> (Min: Int, Max: Int) 
{
    return (Min: 0, Max: 100);
}

point: (Int, Int) = (10, 20);
first := point.Item1;

user := (Id: 1, Name: "Scatty");
userName := user.Name;
```

## Type Equivalence and Assignment

* MUST evaluate tuple type equivalence solely based on the positional sequence of element types.
* MUST ignore element identifiers when evaluating tuple type equivalence or performing assignments.
* MUST permit assignment between tuple instances if and only if their element counts and positional types match identically.

```scatty
pointA: (X: Int, Y: Int) = (10, 20);
pointB: (Width: Int, Height: Int) = (5, 5);
unnamed: (Int, Int) = (0, 0);

pointA = pointB;
unnamed = pointA;
```

## Destructuring

* MAY unpack a tuple into individual bindings using destructuring declaration (`:=`) or destructuring assignment (`=`).
* MUST introduce new local variables into the current scope when destructuring with `:=`.
* MUST assign unpacked values to existing mutable variables when destructuring with `=`.
* MUST require the number of target bindings in a destructuring pattern to match the exact element count of the source tuple.
* MAY use the discard symbol `_` in place of a local identifier to omit an element binding during destructuring.
* MUST NOT generate a variable or allow access to elements bound to `_`.

```scatty
(min, max) := GetBounds();

min = -10;
max = 10;
(min, max) = GetBounds(); 

(id, _) := (Id: 42, Name: "Ignored");
```
