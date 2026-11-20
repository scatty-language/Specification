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
* MUST decrement the ARC `strongCount` (and invoke `Dispose` where applicable) for any materialized temporary Owning Type elements in the caller-allocated stack buffer immediately after the invocation returns, including on error propagation (`throws`) paths.

```scatty
Logger 
{
    // Declares a slice parameter of 'Int'
    // 'metrics' has type '[Int..]'
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

```scatty
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
* MUST declare a variant-constrained receiver (`@::Variant` or `ref @::Variant`) exclusively within a `Data` type that declares `::Variant` or within a `Type with { ... }` extension of that `Data` type.
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

^Valid = Some where @.Value > 0;

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
  3. Decrement the ARC `strongCount` (and invoke `Dispose` where applicable) of the materialized temporary immediately after the call returns, including on error propagation paths.
* MUST lazily materialize a local copy (via bitwise copy for Trivial Types or ARC retain and Copy-on-Write detachment for Owning Types) at the exact point of assignment or mutation if a standard (non-`ref`) parameter is mutated or stored within the method body.
* MUST NOT declare or return references (`ref`) from methods.
* MUST return all `Data` by value.
* For non-`extern` methods:
  * MUST physically return values within CPU registers if the return type's evaluated memory footprint is less than or equal to the target architecture's native pointer size.
  * MUST intrinsically apply Return Value Optimization (RVO) if the return type's evaluated memory footprint exceeds the target architecture's native pointer size.
  * MUST implicitly pass a hidden pointer to a pre-allocated stack block from the caller during RVO.
  * MUST construct and write the returned value directly into the caller's pre-allocated memory during RVO, guaranteeing zero physical memory copying overhead.
* For `extern` methods:
  * MUST pass parameters and return values according to the resolved hardware calling convention defined in the ABI specification.

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

## `extern` Methods

* MAY declare a method with the `extern` modifier (`Method(...) extern [calling_convention] -> ReturnType`).
* MAY specify an explicit calling convention identifier (`system`, `stdcall`, `fastcall`, `vectorcall`, `win64`, or `sysv64`) immediately following the `extern` keyword on a method declaration or `fnptr` type.
* MUST treat the calling convention identifiers (`system`, `stdcall`, `fastcall`, `vectorcall`, `win64`, and `sysv64`) as contextual modifiers valid exclusively after `extern` on methods and `fnptr` types. They MUST NOT be reserved as global keywords.
* MUST NOT specify a calling convention identifier after `extern` on a `Data` declaration.
* MUST compile a method declared with an unqualified `extern` modifier using the target platform's default C calling convention (`cdecl` / `ccc`) instead of the internal `fastcc` calling convention.
* MUST compile a method declared with `extern <calling_convention>` using the specified hardware calling convention defined in the ABI rules.
* MUST reject `extern stdcall` and `extern fastcall` at compile time when the active target architecture is not `x86`.
* MUST reject `extern win64` and `extern sysv64` at compile time when the active target architecture is not `amd64`.
* MUST NOT combine the `extern` modifier (with or without a calling convention specifier) and the `throws` modifier on the same method declaration.
* MUST disable interprocedural register reallocation, parameter reordering, and dead-argument elimination optimizations on methods declared with `extern`.
* MAY pass and return any valid parameter or return type, including non-`extern` `Data` types, in a method declared with `extern`.

```scatty
MathCallbacks 
{
    // Default platform C calling convention ('ccc')
    ComputeDistance(a: Vector, b: Vector) extern -> Float 
    {
        dx := (a.X - b.X) as Float;
        dy := (a.Y - b.Y) as Float;
        return (dx * dx) + (dy * dy);
    }

    // OS system API calling convention ('stdcall' on 32-bit Windows, 'ccc' elsewhere)
    WindowProc(hwnd: Usize, msg: Uint, wParam: Usize, lParam: Size) extern system -> Size 
    {
        return 0;
    }

    // SIMD vector register calling convention
    TransformVector(v: Vector, scale: Float) extern vectorcall -> Vector 
    {
        return Vector { X: (v.X as Float * scale) as Int, Y: (v.Y as Float * scale) as Int };
    }
}
```

## Stack Closures

* MUST represent a closure type (`fn(...) -> ReturnType`) as a stack-bound, non-escaping 2-word fat pointer (`env_ptr` and `code_ptr`) with zero heap allocation.
* MUST NOT allocate heap memory or ARC Control Blocks when constructing, copying, or invoking an `fn` closure.
* MAY declare the `throws` modifier on an `fn` type (`fn(...) throws -> ReturnType`).
* MUST NOT declare the `extern` modifier or calling convention specifiers on an `fn` type.
* MAY assign or pass a non-throwing closure, method, or `fnptr` to an `fn` type declared with `throws`. Compiler MUST emit a forwarding thunk that clears the error indicator register (`swifterror = 0x0`) per the ABI specification.
* MUST NOT assign or pass a `throws` closure, method, or `fnptr` to an `fn` type that omits `throws`.
* MAY implicitly convert a non-`extern` stateless function pointer (`fnptr(...) -> ReturnType`), static method (`Type.StaticMethod`), or unbound instance method (`Type.InstanceMethod`, mapping `@` or `ref @` to the first parameter of `fn`) to a compatible `fn` type.

### Anonymous Functions and Variable Capture

* MUST infer the static type of an anonymous function (`params => body`) as an `fn` closure type unless initialized in a context explicitly expecting an `fnptr` type.
* MAY infer parameter types and the return type of an anonymous function when the expected `fn` or `fnptr` type is statically known from context.
* MUST require explicit parameter type annotations on an anonymous function when bound via type inference (`:=`) without an expected `fn` or `fnptr` type context.
* MAY omit parentheses around the parameter identifier if an anonymous function declares a single non-`ref` parameter with an inferred type (`x => x * 2`).
* MUST require parentheses around the parameter list if the anonymous function declares zero parameters, two or more parameters, a `ref` parameter modifier, or any explicit type annotations.
* MAY declare an anonymous function body as either a single `Expression` or a `{ ... }` block.
* MUST implicitly return the evaluated value of a single `Expression` body.
* MUST require an explicit `return` statement to yield a value from a `{ ... }` block body.
* MAY reference and capture local variables, method parameters, and the `@` receiver from enclosing lexical scopes within an anonymous function typed as `fn`.
* MUST capture any outer local variable, parameter, or `@` receiver that is only read within the anonymous function body as a shared read-only borrow in the stack environment without copying memory or incrementing ARC `strongCount`.
* MUST capture any outer local variable, parameter, or `ref @` receiver that is reassigned, mutated in-place, switched via `=::`, or passed as a `ref` argument within the anonymous function body as an exclusive `ref` borrow in the stack environment.
* MUST govern all captured bindings according to the lexical liveness rules of the Law of Exclusivity.

```scatty
Filter<T>(items: [T..], predicate: fn(T) -> Bool) -> [T] 
{
    item in items -> result := [T].New() 
    {
        where predicate(item);
        result.Add(item);
    }
    return result;
}

ForEach<T>(items: [T..], action: fn(T)) 
{
    item in items => action(item);
}

ProcessScores(scores: [Int..], minScore: Int) 
{
    // Read-only stack capture of 'minScore' (zero heap allocation)
    passed := Filter(scores, s => s >= minScore);

    total := 0;
    history := [String].New();

    // Mutating stack capture of 'total' and 'history' in-place (strongCount remains 1)
    ForEach(passed[..], s => 
    {
        total += s;
        history.Add("Score: \(s)");
    });

    Console.WriteLine("Total: \(total), Entries: \(history.Count)");
}
```

### Bound Instance Methods

* MAY construct a stack closure (`fn`) directly from an instance method access (`instance.Method`) without wrapping it in an anonymous function, binding `instance` as the closure's `env_ptr` and matching the remaining method parameters to the `fn` parameter list.
* MUST treat `instance` as a shared read-only borrow for the duration of the closure's lexical liveness when binding a non-mutating instance method (`@`).
* MUST require `instance` to be a mutable lvalue and treat `instance` as an exclusive `ref` borrow for the duration of the closure's lexical liveness when binding a mutating instance method (`ref @`).
* MUST require `instance` to be statically typed or narrowed to `Data::Variant` or `Type^Guard` when binding a method with a constrained receiver (`@::Variant`, `ref @::Variant`, `@^Guard`, or `ref @^Guard`).

```scatty
Accumulator 
{
    Total: Int = 0;

    Add(ref @, value: Int) => @.Total += value;
    IsAbove(@, value: Int) -> Bool => value > @.Total;
}

ApplyAll(values: [Int..], callback: fn(Int)) 
{
    v in values => callback(v);
}

acc := Accumulator { Total: 10 };
nums := [Int] { 5, 15, 25 };

// Binds 'acc.Add' ('ref @') directly as an 'fn(Int)' stack closure
ApplyAll(nums[..], acc.Add);

// Binds 'acc.IsAbove' ('@') directly as an 'fn(Int) -> Bool' stack closure
filtered := Filter(nums[..], acc.IsAbove);
```

### Non-Escaping Restrictions

* MAY declare an `fn` closure type or an optional closure type (`fn(...)?` / `Option<fn(...)>`) as a method parameter, anonymous function parameter, or local variable.
* MAY pass an `fn` closure type as the type argument to `Option<T>` (`fn(...)?` or `Option<fn(...)>`).
* MUST enforce all non-escaping restrictions and Law of Exclusivity borrow rules of `fn` recursively on `Option<fn(...)>` (and any nested `Option` wrapping it).
* MUST NOT declare an `fn` closure type or `Option<fn(...)>` as:
  * A structural or variant payload field of a `Data` type (excluding the `::Some` payload of `Option<T>` itself),
  * A tuple element,
  * A `static`, `astatic`, or `gstatic` variable or field,
  * A method, anonymous function, or `fnptr` return type (`-> fn(...)` or `-> fn(...)?`), or
  * A type argument to any generic `Data` type other than `Option<T>`, any collection, or any `~Interface`.
* MUST NOT capture an `fn` or `Option<fn(...)>` variable or parameter into a `spawn` or `rollback` block.
* MUST NOT reassign a local `fn` or `Option<fn(...)>` variable to a closure or bound method that captures a variable or instance declared in a narrower inner lexical scope than the variable itself.

```scatty
// Valid: optional non-escaping closure parameter
RunIfPresent(value: Int, callback: fn(Int)? = none) 
{
    may cb := callback => cb(value);
}

// ERROR: 'fn' cannot be returned from a method
MakeAdder(offset: Int) -> fn(Int) -> Int 
{
    return x => x + offset;
}

// ERROR: 'Option<fn(...)>' inherits non-escaping restrictions and cannot be returned
MakeOptionalAdder(offset: Int) -> (fn(Int) -> Int)? 
{
    return .Some(x => x + offset);
}

// ERROR: Cannot reassign outer 'fn' variable to capture narrower-scoped 'temp'
outerFn: (fn(Int) -> Int)? = none;
may condition 
{
    temp := 42;
    outerFn = .Some(x => x + temp);
}
```

## Raw Function Pointers

* MUST represent a raw function pointer type (`fnptr(...) -> ReturnType`) as a stateless, 1-word code address (`sizeof Usize`, `Trivial Type`).
* MAY store `fnptr` values in `Data` fields, tuples, collections, and `static`, `astatic`, or `gstatic` storage, and return `fnptr` values from methods.
* MUST NOT store captured variables, closures, or bound instance state within an `fnptr`.
* MUST reject at compile time any anonymous function initialized as or assigned to an `fnptr` if its body references local variables, parameters, or the `@` receiver from an enclosing lexical scope.
* MAY access the anonymous function's own parameters, compile-time constants (`::`), and `static`, `astatic`, or `gstatic` members within a stateless anonymous function assigned to an `fnptr`.
* MAY declare the `extern` modifier with an optional calling convention specifier on an `fnptr` type (`fnptr(...) extern [calling_convention] -> ReturnType`).
* MUST treat `fnptr(...)` (internal `fastcc`), `fnptr(...) extern` (default C ABI), and each explicitly specified `fnptr(...) extern <calling_convention>` (`system`, `stdcall`, `fastcall`, `vectorcall`, `win64`, `sysv64`) as nominally distinct and mutually incompatible types across all target architectures, even when two conventions lower to the same physical calling convention on the active target.
* MUST match the exact `extern` qualification and calling convention specifier when assigning a method or `fnptr` expression to an `fnptr` type.
* MUST NOT assign a non-`extern` method to an `fnptr(...) extern` pointer, an `extern` method to a non-`extern` `fnptr(...)` pointer, or an `extern` method with one calling convention specifier to an `fnptr` pointer with a different calling convention specifier.
* MAY infer the exact `extern` calling convention on a stateless anonymous function when initialized in a context expecting an `fnptr(...) extern [calling_convention]` type.
* MAY declare the `throws` modifier on a non-`extern` `fnptr` type (`fnptr(...) throws -> ReturnType`).
* MUST NOT combine the `extern` and `throws` modifiers on an `fnptr` type.
* MAY assign a non-throwing stateless method, non-throwing `fnptr`, or non-throwing stateless anonymous function to an `fnptr` type declared with `throws`; the compiler MUST emit a `fastcc` forwarding thunk that clears the error indicator register (`swifterror = 0x0`) and store the thunk's code address in the resulting `fnptr` per the ABI specification (or compile a directly initialized anonymous function with the `throws` ABI).
* MUST NOT assign a `throws` method or throwing anonymous function to an `fnptr` type that omits `throws`.
* MAY assign unbound instance methods (`Type.Method`) to `fnptr` pointers by implicitly mapping the method receiver (`@` or `ref @`) to the first explicitly typed parameter in the `fnptr` signature.
* MUST map a constrained receiver (`@::Variant` or `@^Guard`) to the corresponding constrained parameter type (`Data::Variant` or `Type^Guard`) as the first parameter of the `fnptr` signature.
* MUST perform an indirect call when invoking an `fnptr` pointer.
* MAY devirtualize and inline the invocation if the pointer resolves to a known internal function at compile time.

```scatty
// Standard 'fastcc' stateless function pointer
internalOp: fnptr(Int, Int) -> Int = (a, b) => a + b;

// Default platform C ABI ('ccc') function pointers
cCallback: fnptr(Vector, Vector) extern -> Float = MathCallbacks.ComputeDistance;
cLambda: fnptr(Int) extern -> Int = x => x * 2;

// OS system ABI ('extern system') function pointers
wndProc: fnptr(Usize, Uint, Usize, Size) extern system -> Size = MathCallbacks.WindowProc;
sysLambda: fnptr(Uint) extern system -> Int = code => code as Int;

// SIMD 'vectorcall' function pointer
vecOp: fnptr(Vector, Float) extern vectorcall -> Vector = MathCallbacks.TransformVector;

// Unbound instance method mapped to first parameter of 'fnptr'
mover: fnptr(ref Vector, Int, Int) = Vector.Translate;
processor: fnptr(Bill::Init) -> Bill::Closed = Bill.Process;

// ERROR: Nominal calling convention mismatch ('extern' vs 'extern system')
badProc: fnptr(Vector, Vector) extern system -> Float = MathCallbacks.ComputeDistance;

// ERROR: 'fnptr' cannot capture local variable 'factor'
factor := 10;
badPtr: fnptr(Int) -> Int = x => x * factor;
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
  * Method modifiers (`throws`, `unsafe`, or `extern`) when the parameter signature and return type are identical.
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
* MUST disambiguate candidate overloads that remain tied at the highest parameter and receiver specificity rank and differ by return type (`-> ReturnType`) using the statically known expected return type from the call-site context (including the target type of `as`, `as?`, and `as!` operators, explicit type annotations, or the return type of a target `fn` or `fnptr` type).
* MUST reject the invocation at compile time as ambiguous if two or more candidate overloads remain tied at the highest specificity rank across the supplied arguments and cannot be uniquely disambiguated by a statically known expected return type.
* MUST resolve an overloaded method reference assigned to a stack closure (`fn`) or raw function pointer (`fnptr`) using the explicit parameter types and return type of the target `fn` or `fnptr` type.
* MUST reject assigning an overloaded method identifier to an inferred variable (`:=`) without an explicit `fn` or `fnptr` type annotation.

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

// Disambiguates overloaded method via explicit 'fn' or 'fnptr' type
intChecker: fn(Int) = Validator.Check;
rawChecker: fnptr(Int) = Validator.Check;

s := Scalar { Raw: 42 };
asInt := s as Int;          // Disambiguated by 'as Int' -> calls 'Cast(@) -> Int'
asFloat: Float = s.Cast();  // Disambiguated by expected type 'Float' -> calls 'Cast(@) -> Float'
bad := s.Cast();            // ERROR: Ambiguous overload in inferred ':=' context without expected return type
```
