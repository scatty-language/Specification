# Properties

```EBNF
PropertyDecl ::= Identifier ":" Type ( "=>" Expression ";" | Block )
```

* MUST NOT occupy memory in the Data layout.
* MUST execute the property body upon every access.
* MUST implicitly bind the `@` identifier to the current instance reference within an instance property.
* MUST NOT mutate the `@` instance within the property body.
* MUST treat a property as static if the `@` receiver is not referenced within its body.
* MUST access static properties via the Data type's namespace.

```C#
Vector 
{
    _x: Int;
    _y: Int;

    X: Int => @._x;
    Y: Int => @._y;

    LengthSquared: Int 
    {
        return (@._x * @._x) + (@._y * @._y);
    }

    Zero: Vector => Vector { _x: 0, _y: 0 };
}

vector := Vector.Zero;
x := vector.X;
```
