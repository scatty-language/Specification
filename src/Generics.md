# Generics

* MUST compile generic declarations using monomorphization.
* MUST guarantee at compile time that type arguments satisfy all specified interface constraints.
* MAY declare default type arguments (`T = DefaultType`) or default constant arguments (`N: Type = Expression`) on generic parameters.
* MUST position generic parameters with default values sequentially after all generic parameters lacking default values.

```scatty
Cell<T = Int> 
{
    Value: T;
}

PrinterData<T: ~Printable> 
{
    Item: T;

    PrintItem(@) => @.Item.Print();
}
```

## Constant Generic Parameters

* MAY declare compile-time constant generic parameters within `<...>` using the `Identifier: Type` syntax (`<N: Int>`).
* MUST restrict the type of a constant generic parameter to scalar integer primitives (`Int`, `Uint`, `Long`, `Ulong`, `Size`, `Usize`, `Byte`), `Bool`, `Char`, `Rune`, or a guarded type (`Type^Guard`) over one of these primitive types.
* MAY declare a default compile-time constant expression for a constant generic parameter (`<N: Int = 1>`).
* MUST evaluate all constant generic arguments at compile time.
* MAY supply a literal, a compile-time constant (`::`), an in-scope constant generic parameter, or a parenthesized compile-time constant expression (`(N * 2)`) as a constant generic argument.
* MUST evaluate the governing `must` invariant at compile time when a constant generic parameter is annotated with a type guard (`<N: Int^Positive>`) and reject compilation if the supplied, inferred, or default constant value violates the invariant.
* MUST treat a constant generic parameter within the body of the generic declaration as an immutable compile-time constant (`::`) of its declared type.
* MUST consider two instantiations of a generic type identical if and only if all type arguments match and all constant generic arguments evaluate to the exact same compile-time values.

### Constant Argument Inference

* MAY omit explicit generic arguments when invoking a generic method or instantiating a generic `Data` type if all type and constant generic parameters can be inferred from the call-site arguments or possess default values.
* MUST automatically infer a constant generic parameter `N` from the static type of a supplied argument (such as passing an expression of type `[Int; 4]` to a parameter of type `[Int; N]`).
* MUST automatically infer a constant generic parameter `N` from the element count of a positional collection initializer (`{ expr1, expr2, ... }`) passed to a parameter of type `[T; N]`.
* MUST reject a positional collection initializer Containing a `_ = expr` wildcard fallback when `N` is not explicitly specified or known from an expected type.

```scatty
^Positive = Int must @ > 0;

Matrix<T, Rows: Int^Positive = 4, Cols: Int^Positive = Rows> 
{
    Cells: [T; (Rows * Cols)];
}

PrintNumbers<N: Int>(ints: [Int; N]) 
{
    x in ints => Console.WriteLine(x);
}

// Explicit constant arguments (4x4 matrix of 16 floats)
mat4x4 := Matrix<Float> { Cells: { _ = 0.0f } };

// Explicit rectangular dimensions (2x3 matrix of 6 floats)
mat2x3 := Matrix<Float, 2, 3> { Cells: { _ = 1.0f } };

// ERROR: Violates '^Positive' guard on 'Rows' at compile time
badMat := Matrix<Float, 0, 4> { Cells: { _ = 0.0f } };

// Infers 'N = 3' directly from the 3-element positional initializer
PrintNumbers({ 10, 20, 30 });

// Infers 'N = 2' from the static type of 'pair' ('[Int; 2]')
pair: [Int; 2] = { 1, 2 };
PrintNumbers(pair);
```

## Static Dispatch

* MUST resolve all method calls on constrained generic parameters at compile time as direct, static dispatch calls.

```scatty
ProcessItem<T: ~Entity, ~Printable>(item: T) 
{
    item.Update();
    item.Print();
}
```

## Generic Methods

* MAY declare independent generic type and constant parameters on individual methods.
* MUST NOT permit generic parameters of a method to shadow the generic parameters of the enclosing `Data` declaration.

```scatty
Processor 
{
    Process<T: ~Printable>(item: T) 
    {
        item.Print();
    }
}
```
