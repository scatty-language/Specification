# Properties

Properties provide a concise syntax for computed, read-only data fields.

```EBNF
PropertyDecl ::= Identifier ":" Type ( "=>" Expression ";" | Block )
```

* **No Setters**: To maintain strict predictability memory model, setters are FORBIDDEN.
* **Execution**: A property does not occupy memory in the Data layout. Its body MUST be executed every time it is accessed.
* **Implicit Receiver**: Inside an instance property, the `@` identifier is implicitly available as an immutable view (`@&`) to the current instance. The property CANNOT mutate the instance.
* **Static Properties**: If a property is declared without utilizing the `@` receiver, it accessible via the Data type's namespace.

```C#
Vector 
{
    _x: int;
    _y: int;

    // Expression-bodied property
    X: int => @._x;
    Y: int => @._y;

    // Block-bodied property
    LengthSquared: int 
    {
        return (@._x * @._x) + (@._y * @._y);
    }

    // Static property
    Zero: Vector => Vector { _x: 0, _y: 0 };
}

vector := Vector.Zero; // Static property access
x := vector.X;         // Instance property access
```
