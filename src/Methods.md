# Methods

* MUST NOT declare an explicit void return type.
* MUST omit the `-> ReturnType` clause if no value is returned.
* MAY use the `=>` syntax in place of a block `{}` for single-expression methods.
* MAY declare parameters with default values using the `=` syntax.
* MUST position parameters with default values sequentially after all parameters lacking default values.

## Slice Parameters

* MAY declare the final parameter of a method as a slice parameter by appending the `...` suffix to its element type (`local_id: Type...`).
* MUST position a slice parameter (`local_id: Type...`) as the final parameter in the method signature.
* MUST NOT apply the `ref` modifier or declare a default value (`= Expression`) on a slice parameter.
* MUST statically type a slice parameter (`local_id: Type...`) within the method body as a read-only slice `[Type..]`.
* MUST compile a method declaring `local_id: Type...` as a single function taking `[Type..]` as its final parameter without call-site monomorphization.
* MAY supply zero or more positional arguments of type `Type` to a `Type...` parameter at the call site, or supply a single `[Type..]` (or `[ref Type..]`) slice argument directly.
* MUST NOT use named arguments for individual elements passed to a `Type...` parameter, except when passing a single `[Type..]` slice by parameter name (`local_id: sliceExpr`).
* MUST evaluate positional arguments supplied to a `Type...` parameter into a contiguous caller-allocated temporary stack buffer prior to invocation and pass a `[Type..]` view referencing that stack buffer.
* MUST pass an empty `[Type..]` slice (`data_ptr = 0x0`, `length = 0`) without stack buffer allocation when zero arguments are supplied to a `Type...` parameter.
* MUST decrement the ARC `strongCount` (and invoke `Dispose` where applicable) for any materialized temporary Owning Type elements in the caller-allocated stack buffer immediately after the invocation returns, including on exception propagation (`throws`) paths.

```scatty
Logger 
{
    // Declares a slice parameter of 'Int'; 'metrics' has type '[Int..]'
    LogMetrics(prefix: String, metrics: Int...) 
    {
        m in metrics 
        {
            Console.WriteLine("\(prefix): \(m)");
        }
    }
}

// Call Site 1: 3 positional arguments allocated on the caller's stack
Logger.LogMetrics("A", 10, 20, 30);

// Call Site 2: 0 arguments (passes empty '[Int..]' slice)
Logger.LogMetrics("B");

// Call Site 3: Passing an existing slice directly
history := [Int] { 100, 200, 300 };
Logger.LogMetrics("C", history[..]);
```

## Method Invocation and Arguments

* MUST explicitly bind a named argument (`LocalId: Expression`) to the parameter with the specified identifier.
* MUST bind a positional argument (`Expression`) to the first structurally declared parameter not yet assigned within the current invocation.
* MUST dynamically shift the expected sequence for subsequent positional arguments upon evaluating a named argument.
* MUST intrinsically evaluate and assign the declared default value for any parameter remaining unassigned after all arguments are bound.
* MUST NOT omit an argument for a parameter that lacks a declared default value.
* MUST explicitly include the `ref` modifier at the call site for any argument bound to a parameter declared with `ref`.

```scatty
Process(amount: Int = 1, offset: Int, isDryRun: Bool, force: Bool = false) { }

// Fills in declaration order; 'force' takes its default value (false)
Process(10, 0, true); 

// 'isDryRun' is explicitly assigned
// First positional '10' binds to the first unassigned parameter ('amount')
// Second positional '0' binds to the next unassigned parameter ('offset')
// 'force' remains unassigned and implicitly takes its default value (false)
Process(isDryRun: true, 10, 0);

// 'offset' and 'isDryRun' are explicitly assigned; 'amount' and 'force' take their default values
Process(offset: 0, isDryRun: true);

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
* MUST NOT reassign `@`, switch its active variant via `=::`, mutate any of its fields, pass `@` or its fields as `ref` arguments, or invoke `ref @` methods on `@` within a method declaring an immutable `@` receiver.
* MUST borrow the original instance from the caller for in-place receivers (`ref @`) and apply mutations directly to it.
* MAY reassign `@` via `=` (or `=^` for a `ref @^Guard` receiver) or switch its active variant in-place via `=::` within a method declaring an unconstrained `ref @` receiver.
* MUST NOT switch the active variant of `@` via `=::` or assign a different variant to `@` within a method declaring a variant-constrained receiver (`ref @::Variant`).
* MUST NOT invoke a mutating instance method (`ref @`) on an unbound temporary rvalue expression as a standalone statement where the mutated copy is immediately discarded.

```Scatty
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
* MUST NOT declare a variant-constrained receiver on an untagged union (`tag none`) or within an `~Interface` declaration.
* MUST require the type guard in a guard-constrained receiver (`@^Guard` or `ref @^Guard`) to be applicable to the enclosing `Data` or `~Interface` type.
* MUST statically type `@` as `Data::Variant` within the body of a method declaring `@::Variant` or `ref @::Variant`, exposing both shared members of `Data` and the payload fields of `::Variant`.
* MUST statically type `@` as `Type^Guard` within the body of a method declaring `@^Guard` or `ref @^Guard`.
* MUST require the target receiver expression at the call site to be statically typed or narrowed to `Data::Variant` when invoking a method with an `@::Variant` or `ref @::Variant` receiver.
* MUST require the target receiver expression at the call site to be statically typed or narrowed to `Type^Guard` (or a child guard inheriting from `^Guard`) when invoking a method with an `@^Guard` or `ref @^Guard` receiver.
* MUST reject at compile time any invocation of a constrained-receiver method on an unnarrowed base type, an incompatible variant subtype, or an unguarded type.
* MAY declare the return type of a method as `-> @::Variant` or `-> @^Guard` to synthesize the corresponding variant subtype or guarded type of the enclosing `Data` declaration.

```scatty
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

^Valid = Some must @.Value > 0;

Some with 
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

```scatty
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

```scatty
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

* MUST pass an immutable receiver (`@`) or standard parameter (`param: Type`) within CPU registers if its evaluated memory footprint is less than or equal to the target architecture's native pointer size (`sizeof Usize`).
* MUST pass an immutable receiver (`@`) or standard parameter (`param: Type`) via a hidden read-only pointer under the hood if its evaluated memory footprint exceeds the target architecture's native pointer size (`sizeof Usize`).
* MUST NOT increment or decrement the ARC `strongCount` of an Owning Type (or nested Owning Type fields) upon entering or exiting a method when passed as an immutable `@` receiver or as a read-only parameter, except when call-site value materialization is required.
* MUST perform call-site value materialization prior to method invocation whenever a by-value argument or immutable `@` receiver expression:
  * Is rooted in a `static`, `astatic`, or `gstatic` variable or field (including any chained field access, property evaluation, or index lookup on that static storage), or
  * Reads from a field, property, or indexed element of a root variable that is also bound to a `ref @` receiver or `ref` parameter in the same invocation.
* MUST execute call-site value materialization in the following sequence:
  1. Evaluate the argument expression into an isolated caller-allocated temporary stack slot (via bitwise copy for Trivial Types and ARC `strongCount` increment for Owning Types and nested Owning Type fields) prior to passing any `ref` / `ref @` address or hidden read-only pointer.
  2. Pass the CPU register value or hidden read-only pointer referencing the isolated temporary stack slot to the callee.
  3. Decrement the ARC `strongCount` (and invoke `Dispose` where applicable) of the materialized temporary immediately after the call returns, including on exception propagation paths.
* MUST lazily materialize a local copy (via bitwise copy for Trivial Types or ARC retain and Copy-on-Write detachment for Owning Types) at the exact point of assignment or mutation if a standard (non-`ref`) parameter is mutated or stored within the method body.
* MUST NOT declare or return references (`ref`) from methods.
* MUST return all `Data` by value.
* MUST physically return values within CPU registers if the return type's evaluated memory footprint is less than or equal to the target architecture's native pointer size.
* MUST intrinsically apply Return Value Optimization (RVO) if the return type's evaluated memory footprint exceeds the target architecture's native pointer size.
* MUST implicitly pass a hidden pointer to a pre-allocated stack block from the caller during RVO.
* MUST construct and write the returned value directly into the caller's pre-allocated memory during RVO, guaranteeing zero physical memory copying overhead.

```scatty
// Footprint: 4 bytes (<= 8-byte pointer).
// Returned in a CPU register.
GetId() -> Int 
{
    return 42; 
}

Matrix 
{
    // Footprint: 64 bytes (16 * 4 bytes)
    Cells: [Float; 16]; 
}

// 'mat' (> 8 bytes) is passed via a hidden read-only pointer under the hood;
// zero bytes are copied at the call site because 'mat' is an unaliased local variable.
InspectMatrix(mat: Matrix) -> Float 
{
    return mat.Cells[0];
}

// Guaranteed RVO: 'GetIdentity' writes directly into the caller's stack frame.
GetIdentity() -> Matrix 
{
    return Matrix { Cells: { 0 = 1.0f, _ = 0.0f } };
}
```

## Method Type

* MUST represent a function type (`fn`) as a stateless, raw function pointer (`Trivial Type`).
* MUST NOT store captured variables, closures, or instance state within an `fn` pointer.
* MAY declare the `throws` modifier on an `fn` type (`fn(...) throws -> ReturnType`).
* MAY assign a non-throwing method or anonymous function to an `fn` type declared with `throws`.
* MUST NOT assign a `throws` method or throwing anonymous function to an `fn` type that omits `throws`.
* MAY cast instance methods to `fn` pointers by implicitly mapping the method receiver (`@` or `ref @`) to the first explicitly typed parameter in the `fn` signature.
* MUST map a constrained receiver (`@::Variant` or `@^Guard`) to the corresponding constrained parameter type (`Data::Variant` or `Type^Guard`) as the first parameter of the `fn` signature.
* MUST perform an indirect call when invoking an `fn` pointer.
* MAY devirtualize and inline the invocation if the pointer resolves to a known internal function at compile time.

### Anonymous Functions

* MUST evaluate every anonymous function expression (`params => body`) to a stateless `fn` pointer.
* MUST NOT reference or capture local variables, method parameters, or the `@` receiver from enclosing lexical scopes within an anonymous function body.
* MAY access the anonymous function's own parameters, compile-time constants (`::`), and static members within an anonymous function body.
* MAY infer parameter types and the return type of an anonymous function when the expected `fn` type is statically known from context.
* MUST require explicit parameter type annotations on an anonymous function when bound via type inference (`:=`) without an expected `fn` type context.
* MAY omit parentheses around the parameter identifier if an anonymous function declares a single non-`ref` parameter with an inferred type (`x => x * 2`).
* MUST require parentheses around the parameter list if the anonymous function declares zero parameters, two or more parameters, a `ref` parameter modifier, or any explicit type annotations.
* MAY declare an anonymous function body as either a single `Expression` or a `{ ... }` block.
* MUST implicitly return the evaluated value of a single `Expression` body.
* MUST require an explicit `return` statement to yield a value from a `{ ... }` block body.

```scatty
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

// 2. Pointing to an instance method ('ref @' maps to 'ref Vector')
mover: fn(ref Vector, Int, Int) = Vector.Translate;

vec := factory();
mover(ref vec, 10, 10);

// 3. Stateless anonymous functions coercing to 'fn' pointers
mathOp: fn(Int, Int) -> Int = (a, b) => a + b;
doubleOp: fn(Int) -> Int = x => x * 2;
inferredOp := (a: Int, b: Int) => 
{
    return a + b;
};

// 4. Throwing function pointer
parser: fn(String) throws -> Int = (raw: String) => raw as Int;

// 5. Constrained receivers mapped to first parameter
processor: fn(Bill::Init) -> Bill::Closed = Bill.Process;
validator: fn(Some^Valid) = Some.Process;
```

## Method Overloading

* MAY declare multiple methods with the same identifier within the same `Data` type, interface, or extension block provided their parameter signatures are mutually distinguishable at every possible call site.
* MUST distinguish overloaded methods by:
  * The count of required parameters,
  * The positional types of the parameters,
  * The presence or absence of the `ref` modifier on non-receiver parameters,
  * Distinct receiver type constraints (`@::VariantA` versus `@::VariantB`, or `@^Guard` versus `@`), or
  * Distinct return types (`-> ReturnType`) if and only if each overloaded method satisfies a contract method of a distinct specialization of a generic interface implemented by the enclosing `Data` type.
* MUST NOT overload methods that differ solely by:
  * Receiver mutability (`@` versus `ref @` on the identical receiver type) or instance versus static without a distinct parameter signature,
  * Return type (`-> ReturnType`) on methods that do not satisfy distinct specializations of a generic interface,
  * Parameter identifiers (`(x: Int)` versus `(width: Int)`), or
  * Method modifiers (`throws` or `unsafe`) when the parameter signature and return type are identical.
* MUST reject at declaration time any pair of overloaded methods whose effective parameter signatures (and return types, for generic interface specializations) collide when default parameters or empty slice parameter (`T...`) arguments are omitted.
* MUST NOT overload a method solely by replacing a read-only slice parameter (`[T..]`) with a slice parameter (`T...`) at the same parameter position.
* MUST NOT overload a method solely by the presence or absence of the weak interface contract marker (`Method()` versus `Method?()`).
* MAY implement multiple specializations of a generic interface on the same `Data` type, resolving invocations via standard Overload Resolution rules.

```scatty
Processor 
{
    // Valid overloads: distinct parameter types and 'ref' parameter modifier
    Execute(val: Int) { }
    Execute(ref val: Int) { }
    Execute(val: String) { }

    // ERROR: Cannot overload regular methods solely on return type
    GetValue(@) -> Int => 1;
    GetValue(@) -> String => "one";

    // ERROR: Cannot overload solely on '@' vs 'ref @' receiver
    Reset(@) { }
    Reset(ref @) { }

    // ERROR: Collides with 'Execute(val: Int)' when 'retry' is omitted
    Execute(val: Int, retry: Bool = false) { }

    // ERROR: Collides with 'Execute(val: Int)' when 'extra' is omitted
    Execute(val: Int, extra: Int...) { }

    // ERROR: 'Int...' and '[Int..]' have identical ABI and collide when passed a slice
    Log(items: [Int..]) { }
    Log(items: Int...) { }
}

// Valid: return-type overloading satisfying distinct generic interface specializations
Scalar : ~Castable<Int>, ~Castable<Float>
{
    Raw: Int;

    Cast(@) -> Int => @.Raw;
    Cast(@) -> Float => @.Raw as Float;
}
```

### Overload Resolution Priority

* MUST NOT apply implicit primitive or `~Castable<T>` type conversions during overload resolution.
* MUST select the single best matching overload by evaluating candidate parameters according to the following descending specificity hierarchy:
  1. Exact Type Match: The argument type matches the parameter type identically (e.g., `Int^Positive` matching `Int^Positive`, `[Int; 4]` matching `[Int; 4]`, `Color::Green` matching `Color::Green`, or `[ref Int..]` matching `[ref Int..]`).
  2. Base Type or Read-Only Downgrade: A guarded type (`T^Guard`) or variant subtype (`Data::Variant`) implicitly downgrades to its unguarded base `Data` type `T`, or a mutable slice (`[ref T..]`) implicitly downgrades to a read-only slice (`[T..]`).
  3. Constrained Generic or Guarded Constant Match: The argument matches a generic type parameter constrained by one or more interfaces (`T: ~Interface`) or a constant generic parameter constrained by a type guard (`N: Int^Positive`).
  4. Unconstrained Generic or Unconstrained Constant Match: The argument matches an unconstrained generic type parameter (`T`) or an unconstrained constant generic parameter (`N: Int`).
  5. Slice Parameter Match: Trailing positional arguments are packed into a slice parameter (`T...`).
* MUST apply the same specificity hierarchy to the receiver type when resolving overloaded instance methods (an expression of type `Type^Guard` or `Data::Variant` matches a constrained receiver `@^Guard` or `@::Variant` at Priority 1 over an unconstrained receiver `@` at Priority 2).
* MUST disambiguate candidate overloads that remain tied at the highest parameter and receiver specificity rank and differ by return type (`-> ReturnType`) using the statically known expected return type from the call-site context (including the target type of `as`, `as?`, and `as!` operators, explicit type annotations, or the return type of a target `fn` type).
* MUST reject the invocation at compile time as ambiguous if two or more candidate overloads remain tied at the highest specificity rank across the supplied arguments and cannot be uniquely disambiguated by a statically known expected return type.
* MUST resolve an overloaded method reference assigned to a function pointer (`fn`) using the explicit parameter types and return type of the target `fn` type.
* MUST reject assigning an overloaded method identifier to an inferred variable (`:=`) without an explicit `fn` type annotation.

```scatty
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

s := Scalar { Raw: 42 };
asInt := s as Int;          // Disambiguated by 'as Int' -> calls 'Cast(@) -> Int'
asFloat: Float = s.Cast();  // Disambiguated by expected type 'Float' -> calls 'Cast(@) -> Float'
bad := s.Cast();            // ERROR: Ambiguous overload in inferred ':=' context without expected return type
```
