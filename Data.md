# Data

* MUST be declared at the file's top level.
* MUST evaluate default values prior to initializer execution.
* MUST determine type equality via structural comparison of fields marked with identity modifier, or all fields if no fields marked with identity modifier exist.
* MUST lay out fields sequentially in memory.
* MAY apply memory padding to fields.
* MUST occupy a minimum of 1 byte.

```C#
Vector 
{
    $Id: Int;
    #X: Int;
    #Y: Int = 1;
}

Vector($Id: Int, #X: Int, #Y: Int = 1);
```

## Padding and Bitfields

### Anonymous Padding Fields

* MAY declare anonymous padding fields using the wildcard (`_`).
* MUST NOT be read, mutated, or accessed.
* MUST NOT be required during structural initialization.
* MUST NOT declare default values.
* MUST NOT use declarative modifiers.

### Bitfields

* MAY define exact field bit-width using the Physical Layout operator.
* MUST pack adjacent bitfields into the smallest contiguous memory boundary.
* MUST apply only to scalar integer primitives and `Bool` fields.
* MUST NOT be passed by reference.

```C#
NetworkHeader 
{
    Version: Byte;
    
    _: Byte; 
    _: Byte;
    _: Byte;
    
    IsSYN: Bool ::: 1;
    IsACK: Bool ::: 1;
    _: Byte     ::: 6;
}
```

### Static Fields

* MAY declare a thread-local static field using the `static` modifier after the field's type (`Identifier: Type static = Expression;`).
* MAY declare a process-wide global static field using the `globstatic` modifier after the field's type (`Identifier: Type globstatic = Expression;`).
* MUST allocate `static` field memory in Thread-Local Storage (TLS) so each thread owns an isolated instance of the field.
* MUST allocate `globstatic` field memory exactly once in the global static data segment shared across all threads in the process.
* MAY access and mutate both `static` and `globstatic` fields in safe code without an `unsafe` block.
* MUST NOT access `static` or `globstatic` fields via an instance reference (`@.Field` or `instance.Field`).
* MUST access `static` and `globstatic` fields via the enclosing `Data` type namespace (`DataName.Field`).
* MUST declare a default initialization expression for all `static` and `globstatic` fields.
* MUST evaluate a `static` field's initialization expression once per thread prior to the field's first use on that thread.
* MUST evaluate a `globstatic` field's initialization expression exactly once per process prior to the type's first use.

```C#
Application 
{
    // Thread-Local Storage (TLS): isolated per thread
    ThreadRequests: Int static = 0;

    // Process-wide global storage: shared across all threads
    TotalInstances: Int globstatic = 0;

    Name: String;
    
    New(name: String) -> @ 
    {
        Application.ThreadRequests += 1;
        Application.TotalInstances += 1;
        return Application { Name: name };
    }
}
```

## Variants

* MAY declare mutually exclusive variants alongside standard fields.
* MUST encapsulate a hidden discriminant tag if variants are declared.
* MUST declare at least two variants if any variant is declared.
* MUST allocate standard fields in memory independent of the active variant.
* MAY declare variant-specific payloads.
* MAY omit payload field identifiers.
* MUST automatically assign 1-based positional identifiers (`Item1`, `Item2`, ...) to variant payload fields declared without an explicit identifier.

```C#
Option<T> 
{
    ::None;
    ::Some(T);

    Unwrap(@) throws -> T 
    {
        if @ is .Some(val) => return val;
        throw Exception.New("Unwrapped None");
    }
}
```

### Variants as Subtypes

* MUST treat a variant path (`Data::Variant`) as a compile-time refined subtype of its enclosing `Data` type whose invariant requires the active discriminant tag to match `::Variant`.
* MUST guarantee that a variant subtype (`Data::Variant`) has the exact same physical memory layout as its enclosing `Data` type.
* MUST expose both the shared members of the enclosing `Data` type and the payload fields of `::Variant` directly via member access (`.`) on any expression statically typed as `Data::Variant`.
* MUST infer the enclosing base `Data` type—not the narrowed `Data::Variant` subtype - when initializing a variable with a variant constructor via type inference (`:=`).
* MAY explicitly annotate variables, parameters, or return types with a specific variant subtype (`Data::Variant`).
* MUST NOT assign a different variant to a variable or parameter explicitly typed as a specific variant subtype (`Data::Variant`).
* MAY implicitly downgrade an expression of a variant subtype (`Data::Variant`) to its enclosing base `Data` type without an explicit cast.
* MUST explicitly validate the active discriminant tag using `is`, `as`, `as?`, `as!`, or `match` (or bypass validation via `unas`) when narrowing an expression from a base `Data` type to a specific `Data::Variant` subtype.
* MAY use the contextual leading-dot syntax (`.Variant`) with `is`, `as`, `as?`, `as!`, and `unas` when the base `Data` type is statically known.

```C#
Color 
{
    ::Red(Radius: Float);
    ::Green(X: Int, Y: Int);
}

// Parameter explicitly typed as the 'Color::Green' variant subtype
ProcessGreen(green: Color::Green) 
{
    // Direct access to '::Green' payload fields without pattern matching
    Console.WriteLine(green.X + green.Y);
}

// 'color' is inferred as base type 'Color' via ':='
color := Color::Red(1.0f);

// Can be switched to another variant because 'color' has type 'Color'
color = .Green(10, 20);

// Explicitly typed as 'Color::Red' subtype
strictRed: Color::Red = .Red(5.0f);
Console.WriteLine(strictRed.Radius);

// Implicit downgrade from 'Color::Red' to base type 'Color'
anyColor: Color = strictRed;

// Explicit runtime validation from 'Color' to 'Color::Green'
ProcessGreen(color as Color::Green);
ProcessGreen(color as .Green);

// Flow-sensitive refinement via 'is'
if color is .Green 
{
    Console.WriteLine(color.X);
    ProcessGreen(color);
}
```

### Explicit Tags

* MUST default the implicit variant tag type to `Int` with sequential values starting at `0`.
* MAY explicitly define the tag type and individual tag values.
* MUST resolve the explicit tag type to a scalar integer primitive.
* MUST evaluate explicit variant tag values as compile-time constants.
* MUST assign an omitted variant tag the value of the preceding variant's tag plus `1`.

```C#
HttpResponse : ~Formattable ::: Int 
{
    // Variants
    ::Success ::: 200;
    ::NotFound ::: 404;
    ::Error(Code: Int) ::: 500;

    RequestUrl: String;

    Print(@) => Console.WriteLine("\(@.RequestUrl) failed");
}
```

### Untagged Unions

* MAY remove the discriminant tag by explicitly declaring `::: none`.
* MUST NOT use checked pattern matching or casting operators (`is`, `as`, `as?`, `as!`, `match`) on untagged unions.
* MUST extract payloads or narrow an untagged union to a specific variant subtype using the unchecked cast operator (`unas`).
* MUST enclose any variant declaration containing an Owning Type or `~Disposable` payload within an `unsafe` block or mark the variant declaration with the `unsafe` modifier when the enclosing `Data` type declares `::: none`.
* MUST NOT automatically track, retain, or dispose Owning Type payloads inside an untagged union (`::: none`); shared fields declared outside the variants remain automatically managed via ARC.

```C#
FloatIntUnion ::: none 
{
    ::FloatBits(Float);
    ::IntBits(Int);
}

RawBufferUnion ::: none 
{
    ::RawPtr(Usize);

    unsafe 
    {
        ::ManagedText(String);
        ::ManagedList([+ Int]);
    }

    // Shared field outside the union remains safely tracked by ARC
    Label: String;
}

myUnion := FloatIntUnion::IntBits(42);
floatVal := myUnion unas .FloatBits;
```
