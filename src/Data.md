# Data

* MUST be declared at the file's top level.
* MUST evaluate default values prior to initializer execution.
* MUST determine type equality via structural comparison of fields marked with the `equality` modifier, or all fields if no fields marked with `equality` exist.
* MUST lay out fields sequentially in memory.
* MAY apply memory padding to fields.
* MUST occupy a minimum of 1 byte.

```scatty
Vector 
{
    Id: Int equality;
    X: Int math;
    Y: Int math = 1;
}

Vector(Id: Int equality, X: Int math, Y: Int math = 1);
```

## Padding and Bitfields

### Anonymous Padding Fields

* MAY declare anonymous padding fields using the wildcard (`_`).
* MUST NOT be read, mutated, or accessed.
* MUST NOT be required during structural initialization.
* MUST NOT declare default values.
* MUST NOT use declarative modifiers (`equality`, `math`).

### Bitfields

* MAY define exact field bit-width using the `bitwidth` modifier (`Field: Type bitwidth N;`).
* MUST pack adjacent bitfields into the smallest contiguous memory boundary.
* MUST apply only to scalar integer primitives and `Bool` fields.
* MUST NOT be passed by reference.

```scatty
NetworkHeader 
{
    Version: Byte;
    
    _: Byte; 
    _: Byte;
    _: Byte;
    
    IsSYN: Bool bitwidth 1;
    IsACK: Bool bitwidth 1;
    _: Byte     bitwidth 6;
}
```

### Static Fields

* MAY declare a thread-local static field using the `static` modifier after the field's type (`Identifier: Type static = Expression;`).
* MAY declare an atomic process-wide global static field using the `astatic` modifier after the field's type (`Identifier: Type astatic = Expression;`).
* MAY declare a raw unsynchronized process-wide global static field using the `gstatic` modifier after the field's type (`Identifier: Type gstatic = Expression;`).
* MUST allocate `static` field memory in Thread-Local Storage (TLS) so each thread owns an isolated instance of the field.
* MUST allocate `astatic` and `gstatic` field memory once in the global static data segment shared across all threads in the process.
* MAY access and mutate `static` and `astatic` fields in safe code without an `unsafe` block.
* MUST compile reads, writes, and compound assignments on Trivial Type `astatic` fields whose memory footprint is 64 bits or less as lock-free hardware atomic operations.
* MUST synchronize reads, writes, field mutations, and mutating method calls (`ref @`) on Owning Type `astatic` fields and Trivial Type `astatic` fields exceeding 64 bits via an implicit per-field atomic spinlock.
* MUST acquire the per-field atomic spinlock, increment `strongCount` (for Owning Types) or copy the memory (for Trivial Types), and release the spinlock when reading a spinlock-backed `astatic` field.
* MUST hold the per-field atomic spinlock for the entire read-modify-write sequence when executing a compound assignment, field mutation, variant switch (`=::`), or mutating method call (`ref @`) on a spinlock-backed `astatic` field, releasing any replaced Owning Type reference after unlocking the spinlock.
* MUST restrict all reads, mutations, and references to `gstatic` fields exclusively to `unsafe` blocks or `unsafe` methods.
* MUST NOT pass a `static` or `astatic` field directly as a `ref` argument to a method parameter.
* MUST NOT access `static`, `astatic`, or `gstatic` fields via an instance reference (`@.Field` or `instance.Field`).
* MUST access `static`, `astatic`, and `gstatic` fields via the enclosing `Data` type namespace (`DataName.Field`).
* MUST declare a default initialization expression for all `static`, `astatic`, and `gstatic` fields.
* MUST evaluate a `static` field's initialization expression once per thread prior to the field's first use on that thread.
* MUST evaluate an `astatic` or `gstatic` field's initialization expression exactly once per process prior to the type's first use.

```scatty
Application 
{
    // Thread-Local Storage (TLS): isolated per thread, safe
    ThreadRequests: Int static = 0;

    // Atomic process-wide storage: shared across threads, safe atomic access
    TotalInstances: Int astatic = 0;

    // Raw unsynchronized global storage: requires 'unsafe'
    RawHandle: Usize gstatic = 0;

    Name: String;
    
    New(name: String) -> @ 
    {
        Application.ThreadRequests += 1;
        Application.TotalInstances += 1;

        unsafe 
        {
            Application.RawHandle = 0xFF00;
        }

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

```scatty
Option<T> 
{
    ::None;
    ::Some(T);

    Unwrap(@) throws -> T 
    {
        may @ is .Some(val) => return val;
        throw Exception.New("Unwrapped None");
    }
}
```

### Variants as Subtypes

* MUST treat a variant path (`Data::Variant`) as a compile-time guarded subtype of its enclosing `Data` type whose invariant requires the active discriminant tag to match `::Variant`.
* MUST guarantee that a variant subtype (`Data::Variant`) has the exact same physical memory layout as its enclosing `Data` type.
* MUST expose both the shared members of the enclosing `Data` type and the payload fields of `::Variant` directly via member access (`.`) on any expression statically typed as `Data::Variant`.
* MUST infer the enclosing base `Data` type—not the narrowed `Data::Variant` subtype - when initializing a variable with a variant constructor via type inference (`:=`).
* MAY explicitly annotate variables, parameters, or return types with a specific variant subtype (`Data::Variant`).
* MUST NOT assign a different variant to a variable or parameter explicitly typed as a specific variant subtype (`Data::Variant`).
* MAY implicitly downgrade an expression of a variant subtype (`Data::Variant`) to its enclosing base `Data` type without an explicit cast.
* MUST explicitly validate the active discriminant tag using `is`, `as`, `as?`, `as!`, or `match` (or bypass validation via `unas`) when narrowing an expression from a base `Data` type to a specific `Data::Variant` subtype.
* MAY use the contextual leading-dot syntax (`.Variant`) with `is`, `as`, `as?`, `as!`, and `unas` when the base `Data` type is statically known.
* MAY constrain the receiver of an instance method to a specific variant subtype (`@::Variant` or `ref @::Variant`).
* MUST expose the payload fields of `::Variant` directly on `@` within the body of a method declaring an `@::Variant` or `ref @::Variant` receiver.
* MUST reject invoking a method declared with `@::Variant` or `ref @::Variant` unless the receiver expression is statically typed or narrowed to `Data::Variant`.
* MUST NOT apply the variant switch operator (`=::`) or assign a different variant to a variable, field, or parameter while it is statically typed or narrowed to a specific variant subtype (`Data::Variant`).
* MAY pass an expression statically typed or narrowed to `Data::Variant` as a `ref` argument to a `ref Data` parameter (or invoke an enclosing `ref @` method on it) if and only if the target method is statically known and does not:
  * Apply the `=::` operator to the `ref` binding,
  * Reassign the entire `ref` binding via `=`, or
  * Forward the `ref` binding to another method or `fn` pointer that may mutate the active variant.
* MUST reject at compile time passing a `Data::Variant` expression by `ref` to any `ref Data` method that mutates the active variant or to an indirect function pointer (`fn(ref Data)`).

```scatty
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

// Flow-sensitive type guard via 'is'
may color is .Green 
{
    Console.WriteLine(color.X);
    ProcessGreen(color);
}

Order 
{
    ::Draft(Items: [String]);
    ::Submitted(TransactionId: Int);

    // Direct access to '::Draft' payload ('@.Items') without pattern matching
    AddItem(ref @::Draft, item: String) 
    {
        @.Items.Add(item);
    }

    Submit(@::Draft, txId: Int) -> @::Submitted 
    {
        return .Submitted(txId);
    }
}

// Explicitly typed as 'Order::Draft'
draft: Order::Draft = .Draft([String].New());
draft.AddItem("Book");
submitted: Order::Submitted = draft.Submit(1001);

// Inferred as base type 'Order'
order := Order::Draft([String].New());
order.AddItem("Pen"); // ERROR: 'order' has type 'Order', not 'Order::Draft'

// Valid after flow-sensitive narrowing via 'is'
may order is .Draft 
{
    order.AddItem("Pen");
}

Session 
{
    ::Disconnected;
    ::Connected(SocketId: Int);

    Retries: Int = 0;

    // Safe for 'Session::Connected': mutates only shared field 'Retries'
    IncrementRetries(ref @) 
    {
        @.Retries += 1;
    }

    // Mutates active variant via '=::'
    Reset(ref @) 
    {
        @ =:: Disconnected;
    }
}

active: Session::Connected = .Connected(42);

// Valid: 'IncrementRetries' does not alter the active variant
active.IncrementRetries();

// ERROR: 'active' is typed as 'Session::Connected'; cannot switch variant directly
active =:: Disconnected;

// ERROR: 'Reset' mutates the active variant via '=::' and cannot be called on 'Session::Connected'
active.Reset();

// Valid after widening to the base 'Session' type
anySession: Session = active;
anySession =:: Disconnected;
anySession.Reset();
```

### Explicit Tags

* MUST default the implicit variant tag type to `Int` with sequential values starting at `0`.
* MAY explicitly define the tag type using the `tag` keyword (`Data tag Type`) and individual variant tag values using `=` (`::Variant = Expression;`).
* MUST resolve the explicit tag type to a scalar integer primitive.
* MUST evaluate explicit variant tag values as compile-time constants.
* MUST assign an omitted variant tag the value of the preceding variant's tag plus `1`.

```scatty
HttpResponse : ~Formattable tag Int 
{
    // Variants
    ::Success = 200;
    ::NotFound = 404;
    ::Error(Code: Int) = 500;

    RequestUrl: String;

    Print(@) => Console.WriteLine("\(@.RequestUrl) failed");
}
```

### Untagged Unions

* MAY remove the discriminant tag by explicitly declaring `tag none`.
* MUST NOT use checked pattern matching or casting operators (`is`, `as`, `as?`, `as!`, `match`) on untagged unions.
* MUST extract payloads or narrow an untagged union to a specific variant subtype using the unchecked cast operator (`unas`).
* MUST enclose any variant declaration containing an Owning Type or `~Disposable` payload within an `unsafe` block or mark the variant declaration with the `unsafe` modifier when the enclosing `Data` type declares `tag none`.
* MUST NOT automatically track, retain, or dispose Owning Type payloads inside an untagged union (`tag none`); shared fields declared outside the variants remain automatically managed via ARC.

```scatty
FloatIntUnion tag none 
{
    ::FloatBits(Float);
    ::IntBits(Int);
}

RawBufferUnion tag none 
{
    ::RawPtr(Usize);

    unsafe 
    {
        ::ManagedText(String);
        ::ManagedList([Int]);
    }

    // Shared field outside the union remains safely tracked by ARC
    Label: String;
}

myUnion := FloatIntUnion::IntBits(42);
floatVal := myUnion unas .FloatBits;
```
