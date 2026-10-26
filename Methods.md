# Methods

* MUST NOT declare an explicit void return type.
* MUST omit the `-> ReturnType` clause if no value is returned.
* MAY use the `=>` syntax in place of a block `{}` for single-expression methods.
* MAY declare parameters with default values using the `=` syntax.
* MUST position parameters with default values sequentially after all parameters lacking default values.

## Variadic Parameters

* MUST position a variadic parameter (`LocalId: ...Type`) as the final parameter in the method signature.
* MUST NOT declare a default value for a variadic parameter.
* MUST intrinsically evaluate the variadic parameter within the method body as a Static Array (`[Type ::: N]`), where `N` equals the exact number of passed arguments.
* MUST generate a unique, distinct function signature during monomorphization for every distinct argument count evaluated at call sites.
* MUST supply arguments to a variadic parameter positionally.
* MUST NOT use named arguments for variadic elements.
* MUST intrinsically generate and pass an empty static array (`[Type ::: 0]`) if no arguments are provided to the variadic parameter.

```C#
Logger 
{
    // Declares a variadic parameter for 'Int'
    LogMetrics(prefix: String, metrics: ...Int) 
    {
        // 'metrics' acts as a Static Array ([Int ::: N])
        m in metrics 
        {
            Console.WriteLine(m);
        }
    }
}

// Call Site 1: 3 variadic arguments
// Compiler generates: LogMetrics_3(prefix: String, metrics: [Int ::: 3])
Logger.LogMetrics("A", 10, 20, 30);

// Call Site 2: 2 variadic arguments
// Compiler generates: LogMetrics_2(prefix: String, metrics: [Int ::: 2])
Logger.LogMetrics("B", 5, 10);

// Call Site 3: 0 variadic arguments
// Compiler generates: LogMetrics_0(prefix: String, metrics: [Int ::: 0])
Logger.LogMetrics("C");
```

## Method Invocation and Arguments

* MUST explicitly bind a named argument (`LocalId: Expression`) to the parameter with the specified identifier.
* MUST bind a positional argument (`Expression`) to the first structurally declared parameter not yet assigned within the current invocation.
* MUST dynamically shift the expected sequence for subsequent positional arguments upon evaluating a named argument.
* MUST intrinsically evaluate and assign the declared default value for any parameter remaining unassigned after all arguments are bound.
* MUST explicitly include the `ref` modifier at the call site for any argument bound to a parameter declared with `ref`.

```C#
Process(amount: Int, offset: Int, isDryRun: Bool, force: Bool = false) { }

// Fills in declaration order
// Required type order: 'Int', 'Int', 'Bool'
Process(10, 0, true); 

// 'isDryRun' is explicitly assigned
// First positional '10' binds to the first unassigned parameter ('amount')
// Second positional '0' binds to the next unassigned parameter ('offset')
// 'force' remains unassigned and implicitly takes its default value (false)
Process(isDryRun: true, 10, 0);

Update(ref target: Vector, x: Int, y: Int) { }

vec := Vector.New(0, 0);

// 'y' is named
// Positional 'ref vec' binds to 'target'
// Positional '5' binds to 'x'
Update(y: 10, ref vec, 5); 
```

## Instance Methods

* MUST declare the receiver (`@` or `ref @`) as the first parameter.
* MUST treat an unqualified `@` receiver as an immutable reference to the caller's instance.
* MUST NOT mutate any fields of `@`, pass `@` or its fields as `ref` arguments, or invoke `ref @` methods on `@` within a method declaring an immutable `@` receiver.
* MUST borrow the original instance from the caller for in-place receivers (`ref @`) and apply mutations directly to it.
* MUST NOT reassign the `@` or `ref @` receiver binding itself.
* MUST NOT invoke a mutating instance method (`ref @`) on an unbound temporary rvalue expression as a standalone statement where the mutated copy is immediately discarded.

```C#
Vector
{
    X: Int;
    Y: Int;

    New(x: Int, y: Int) -> @ => Vector { X: x, Y: y };

    // '@' is an immutable view of the original instance (zero copy, zero ARC overhead)
    Sum(@, other: Vector) -> Vector 
    {
        return Vector.New(@.X + other.X, @.Y + other.Y);
    }

    // Returns a modified copy using 'with' while keeping '@' immutable
    WithX(@, x: Int) -> @ => @ with { X: x };

    // Expression body, reads the immutable receiver
    Print(@) => Console.WriteLine("\(@.X), \(@.Y)");

    // 'ref @' mutates the caller's original instance in-place
    Translate(ref @, dx: Int, dy: Int) 
    {
        @.X += dx;
        @.Y += dy;
    }
}
```

### Constrained Receivers

* MAY constrain an instance receiver to a specific variant subtype (`@::Variant` or `ref @::Variant`) or a type guard (`@^Guard` or `ref @^Guard`).
* MUST declare a variant-constrained receiver (`@::Variant` or `ref @::Variant`) exclusively within a `Data` type that declares `::Variant` or within a `+Type` extension of that `Data` type.
* MUST NOT declare a variant-constrained receiver on an untagged union (`::: none`) or within an `~Interface` declaration.
* MUST require the type guard in a guard-constrained receiver (`@^Guard` or `ref @^Guard`) to be applicable to the enclosing `Data` or `~Interface` type.
* MUST statically type `@` as `Data::Variant` within the body of a method declaring `@::Variant` or `ref @::Variant`, exposing both shared members of `Data` and the payload fields of `::Variant`.
* MUST statically type `@` as `Type^Guard` within the body of a method declaring `@^Guard` or `ref @^Guard`.
* MUST require the target receiver expression at the call site to be statically typed or narrowed to `Data::Variant` when invoking a method with an `@::Variant` or `ref @::Variant` receiver.
* MUST require the target receiver expression at the call site to be statically typed or narrowed to `Type^Guard` (or a child guard inheriting from `^Guard`) when invoking a method with an `@^Guard` or `ref @^Guard` receiver.
* MUST reject at compile time any invocation of a constrained-receiver method on an unnarrowed base type, an incompatible variant subtype, or an unguarded type.
* MAY declare the return type of a method as `-> @::Variant` or `-> @^Guard` to synthesize the corresponding variant subtype or guarded type of the enclosing `Data` declaration.

```C#
Bill 
{
    ::Init;
    ::Closed;

    Process(@::Init) -> Bill::Closed 
    {
        return .Closed;
    }
}

Some 
{
    Value: Int;
}

^Valid = Some where @.Value > 0;

+Some 
{
    Process(@^Valid) 
    {
        Console.WriteLine(@.Value);
    }
}

// ERROR: 'bill' is inferred as base type 'Bill' (and holds '::Closed'), not 'Bill::Init'
bill := Bill::Closed;
bill.Process();

// Valid: explicitly typed as 'Bill::Init'
initBill: Bill::Init = .Init;
closedBill: Bill::Closed = initBill.Process();

// ERROR: 'some' is unguarded 'Some', not 'Some^Valid'
some := Some { Value: 10 };
some.Process();

// Valid: cast to 'Some^Valid' prior to invocation
validSome := some as Some^Valid;
validSome.Process();
```

## Static Methods

* MUST classify methods lacking an `@` receiver parameter as static methods.
* MUST NOT access instance fields from within a static method.
* MUST invoke static methods using the type identifier as a namespace qualifier (`DataName.MethodName()`).
* MAY omit the type identifier qualifier when invoking a static method from within the lexical scope of its enclosing type.
* MAY declare the return type of a static method as `-> @` to synthesize the explicit return type of the enclosing `Data` declaration.

```C#
Vector 
{
    // Static method returning the explicitly named type
    CreateEmpty() -> Vector 
    {
        return Vector { X: 0, Y: 0 };
    }

    // Static method using the '@' shorthand for the return type
    New(x: Int, y: Int) -> @
    {
        return Vector { X: x, Y: y };
    }

    // Unqualified static method call
    CreateZero() -> @
    {
        return New(0, 0);
    }
}

zero := Vector.CreateZero();
```

## Method Cascading (`..`)

* MAY chain sequential method invocations onto the same receiver instance using the cascade operator (`..`) following an initial member method invocation (`.`).
* MUST resolve the target receiver of a cascade chain (`target.Method1()..Method2()..Method3()`) to the receiver expression immediately preceding the initial `.Method1()` invocation.
* MUST evaluate the receiver expression exactly once and invoke each cascaded method sequentially on that receiver instance.
* MUST discard any return values produced by intermediate methods within a cascade chain.
* MUST restrict cascade chains to statement-level contexts:
  * Standalone cascade statements (`target.Method1()..Method2();`).
  * The top-level right-hand side of a variable declaration (`x := Init().Method1()..Method2();`).
  * The top-level right-hand side of an assignment statement (`x = Init().Method1()..Method2();`).
  * The top-level expression of a `return` statement (`return Init().Method1()..Method2();`).
* MUST NOT embed a cascade chain inside sub-expressions, binary operations, or method argument lists.
* MUST require the target receiver of a standalone (unassigned) cascade statement to be a mutable lvalue if any method in the cascade chain requires an in-place receiver (`ref @`).
* MUST desugar a cascade chain bound to a variable, assignment target, or `return` statement by initializing the target with the evaluated receiver expression first and subsequently executing each method in the chain on that target.

```C#
Vector 
{
    X: Int;
    Y: Int;

    New(x: Int, y: Int) -> @ => Vector { X: x, Y: y };

    Add(ref @, other: Vector) 
    {
        @.X += other.X;
        @.Y += other.Y;
    }
}

pos := Vector.New(0, 0);

// Standalone cascade statement on an lvalue variable:
// Invokes 'Add' three times sequentially on 'pos'
pos.Add(.New(5, 0))..Add(.New(0, 5))..Add(.New(0, 10));

// Cascade initialization of a new variable from an rvalue call:
// Desugars into:
//   nextPos := GetActivePosition();
//   nextPos.Add(a);
//   nextPos.Add(b);
nextPos := GetActivePosition().Add(a)..Add(b);

// Cascade on a nested property path:
// Evaluates 'app.Window.Canvas' once and invokes all three methods on that Canvas
app.Window.Canvas.Clear()..DrawLine(0, 0, 10, 10)..Present();

// ERROR: Cannot mutably cascade on an unbound rvalue temporary without assigning the result
GetActivePosition().Add(a)..Add(b);

// ERROR: Cascade chains cannot be nested inside method arguments
ProcessVector(pos.Add(a)..Add(b));
```

## Parameter Passing and Return Semantics

* MUST pass an immutable receiver (`@`) or standard parameter (`param: Type`) within CPU registers if its evaluated memory footprint is less than or equal to the target architecture's native pointer size (`sizeof(Usize)`).
* MUST pass an immutable receiver (`@`) or standard parameter (`param: Type`) via a hidden pointer under the hood if its evaluated memory footprint exceeds the target architecture's native pointer size (`sizeof(Usize)`), avoiding eager memory copying at the call site.
* MUST NOT increment or decrement the ARC `strongCount` of an Owning Type upon entering or exiting a method when passed as an immutable `@` receiver or as a read-only parameter.
* MUST lazily materialize a local copy (via bitwise copy for Trivial Types or ARC retain and Copy-on-Write detachment for Owning Types) at the exact point of assignment or mutation if a standard (non-`ref`) parameter is mutated or stored within the method body.
* MUST NOT declare or return references (`ref`) from methods.
* MUST return all `Data` by value.
* MUST physically return values within CPU registers if the return type's evaluated memory footprint is less than or equal to the target architecture's native pointer size.
* MUST intrinsically apply Return Value Optimization (RVO) if the return type's evaluated memory footprint exceeds the target architecture's native pointer size.
* MUST implicitly pass a hidden pointer to a pre-allocated stack block from the caller during RVO.
* MUST construct and write the returned value directly into the caller's pre-allocated memory during RVO, guaranteeing zero physical memory copying overhead.

```C#
// Footprint: 4 bytes (<= 8-byte pointer).
// Returned in a CPU register.
GetId() -> Int 
{
    return 42; 
}

Matrix 
{
    // Footprint: 64 bytes (16 * 4 bytes)
    Cells: [Float ::: 16]; 
}

// 'mat' (> 8 bytes) is passed via a hidden read-only pointer under the hood;
// zero bytes are copied at the call site because 'mat' is only read.
InspectMatrix(mat: Matrix) -> Float 
{
    return mat.Cells[0];
}

// Guaranteed RVO: 'GetIdentity' writes directly into the caller's stack frame.
GetIdentity() -> Matrix 
{
    return Matrix { Cells: [Float ::: 16] { 1.0f, _ = 0.0f } };
}
```

## Method Type

* MUST represent a function type (`fn`) as a stateless, raw function pointer.
* MUST NOT store captured variables, closures, or instance state within an `fn` pointer.
* MAY cast instance methods to `fn` pointers by implicitly mapping the method receiver (`@` or `ref @`) to the first explicitly typed parameter in the `fn` signature.
* MUST map a constrained receiver (`@::Variant` or `@^Guard`) to the corresponding constrained parameter type (`Data::Variant` or `Type^Guard`) as the first parameter of the `fn` signature.
* MUST perform an indirect call when invoking an `fn` pointer.
* MAY devirtualize and inline the invocation if the pointer resolves to a known internal function at compile time.

```C#
Vector 
{
    X: Int;
    Y: Int;

    // Static method
    Create() -> Vector => Vector { X: 0, Y: 0 };

    // Instance method
    Translate(ref @, dx: Int, dy: Int) 
    {
        @.X += dx;
        @.Y += dy;
    }
}

// 1. Pointing to a static method
factory: fn() -> Vector = Vector.Create;

// 2. Pointing to an instance method
// The 'ref @' receiver is structurally treated as 'ref Vector'
mover: fn(ref Vector, Int, Int) = Vector.Translate;

vec := factory();
mover(ref vec, 10, 10);

// 3. Stateless lambda coercing to a raw function pointer
mathOp: fn(Int, Int) -> Int = (a, b) => a + b;

// Maps '@::Init' receiver to 'Bill::Init' first parameter
processor: fn(Bill::Init) -> Bill::Closed = Bill.Process;

// Maps '@^Valid' receiver to 'Some^Valid' first parameter
validator: fn(Some^Valid) = Some.Process;
```

## Method Overloading

* MAY declare multiple methods with the same identifier within the same `Data` type, interface, or extension block provided their parameter signatures are mutually distinguishable at every possible call site.
* MUST distinguish overloaded methods by:
  * The count of required parameters,
  * The positional types of the parameters,
  * The presence or absence of the `ref` modifier on non-receiver parameters, or
  * Distinct receiver type constraints (`@::VariantA` versus `@::VariantB`, or `@^Guard` versus `@`).
* MUST NOT overload methods that differ solely by:
  * Receiver mutability (`@` versus `ref @` on the identical receiver type) or instance versus static without a distinct parameter signature,
  * Return type (`-> ReturnType`),
  * Parameter identifiers (`(x: Int)` versus `(width: Int)`), or
  * Method modifiers (`throws` or `unsafe`).
* MUST reject at declaration time any pair of overloaded methods whose effective parameter signatures collide when trailing default parameters or empty variadic arguments are omitted.
* MUST NOT overload a method solely by the presence or absence of the weak interface contract marker (`Method()` versus `Method?()`).
* MAY implement multiple specializations of a generic interface on the same `Data` type, resolving invocations via standard Overload Resolution rules.

```C#
Processor 
{
    // Valid overloads: distinct parameter types and 'ref' parameter modifier
    Execute(val: Int) { }
    Execute(ref val: Int) { }
    Execute(val: String) { }

    // ERROR: Cannot overload solely on '@' vs 'ref @' receiver
    Reset(@) { }
    Reset(ref @) { }

    // ERROR: Collides with 'Execute(val: Int)' when 'retry' is omitted
    Execute(val: Int, retry: Bool = false) { }
}
```

### Overload Resolution Priority

* MUST NOT apply implicit primitive or `~Castable<T>` type conversions during overload resolution.
* MUST select the single best matching overload by evaluating candidate parameters according to the following descending specificity hierarchy:
  1. Exact Type Match: The argument type matches the parameter type identically (e.g., `Int^Positive` matching `Int^Positive`, or `Color::Green` matching `Color::Green`).
  2. Base Type Downgrade: A guarded type (`T^Guard`) or variant subtype (`Data::Variant`) implicitly downgrades to its unguarded base `Data` type `T`.
  3. Constrained Generic Match: The argument matches a generic type parameter constrained by one or more interfaces (`T: ~Interface`).
  4. Unconstrained Generic Match: The argument matches an unconstrained generic type parameter (`T`).
  5. Variadic Parameter Match: Trailing arguments are packed into a variadic parameter (`...T`).
* MUST reject the invocation at compile time as ambiguous if two or more candidate overloads remain tied at the highest specificity rank across the supplied arguments.
* MUST resolve an overloaded method reference assigned to a function pointer (`fn`) using the explicit parameter types of the target `fn` type.
* MUST apply the same specificity hierarchy to the receiver type when resolving overloaded instance methods (an expression of type `Type^Guard` or `Data::Variant` matches a constrained receiver `@^Guard` or `@::Variant` at Priority 1 over an unconstrained receiver `@` at Priority 2).
* MUST reject assigning an overloaded method identifier to an inferred variable (`:=`) without an explicit `fn` type annotation.

```C#
Validator 
{
    // Priority 1 for 'Int^Positive'
    Check(val: Int^Positive) => Console.WriteLine("Verified positive");

    // Priority 2 for 'Int^Positive' (via base downgrade), Priority 1 for plain 'Int'
    Check(val: Int) => Console.WriteLine("Plain integer");

    // Priority 4 (Unconstrained generic)
    Check<T>(val: T) => Console.WriteLine("Generic fallback");
}

raw: Int = -5;
pos: Int^Positive = 10 as ^;

Validator.Check(pos);   // Calls 'Check(val: Int^Positive)' (Exact Match)
Validator.Check(raw);   // Calls 'Check(val: Int)' (Exact Match)
Validator.Check("Hi");  // Calls 'Check<T>(val: T)' (Generic Match)

// Disambiguates overloaded method via explicit 'fn' type
intChecker: fn(Int) = Validator.Check;
```
