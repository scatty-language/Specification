# Error Handling

* MUST require types thrown as errors to be `Data` types implementing the `~Error` standard interface.
* MAY omit the weak `Message?: String` contract property when implementing `~Error` on a concrete `Data` type.

```scatty
~Error
{
    Message?: String;
}
```

## Potentially Throwing Operations

* MUST classify the following operations as potentially throwing operations across all build configurations:
  * Explicit `throw` statements.
  * Invocations of methods, stack closures (`fn`), or raw function pointers (`fnptr`) declared with the `throws` modifier.
  * Fallible `as` conversions (`~Castable<T>` implementations declaring `throws`, runtime type guard casts `as Type^Guard`, and runtime variant subtype casts `as Data::Variant`), which raise `CastError` or the `Cast` method's error upon failure.
  * Fallible indexing (`[]`) and slice range indexing (`[start..end]`) operations.
  * Runtime integer division (`/`) and remainder (`%`) operations with a non-constant divisor (raising `DivisionByZeroError` on a zero divisor, or `OverflowError` on signed `MinValue / -1` when overflow checks are active).
  * Runtime integer arithmetic operations (`+`, `-`, `*`, and unary `-`) on non-constant operands (raising `OverflowError` when overflow checks are active).
  * Runtime heap allocation operations—including constructing an Owning Type (`~Disposable`), Copy-Update (`with`) on an Owning Type, Copy-on-Write buffer detachment, runtime `String` interpolation, `String` / `RuneString` conversions, and dynamic heap collection allocations or resizes (raising `OutOfMemoryError` upon allocator failure).
  * Guarded mutations (`=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`, guarded field mutations, post-`ref` / post-`ref @` invariant re-evaluations, and post-`ref`-stage loop exit checks) on non-constant targets (raising `InvariantError` when invariant checks are active).
* MUST NOT classify compile-time constant expressions proven not to fail at compile time as potentially throwing operations.

## The `throws` Modifier

* MUST classify methods lacking the `throws` modifier as non-throwing.
* MUST NOT execute a `throw Expression;` statement within a non-throwing method unless enclosed within a `try` / `catch` construct that intercepts it.
* MAY execute potentially throwing operations within a non-throwing method without a `try` construct.
* MUST immediately terminate the process via Abort or Fatal Panic mechanics if an uncaught error escapes any potentially throwing operation within a non-throwing method.
* MAY execute `throw Expression;` statements within methods declared with the `throws` modifier.
* MUST propagate uncaught errors to the calling scope in methods declared with the `throws` modifier.

```scatty
TryCalculate(value: Int) throws
{
    may value < 0 => throw Error.New("Less than zero!");
}

Calculate()
{
    // Valid: handled via try/catch
    try
    {
        TryCalculate(-1);
    }
    catch
    {
        Console.WriteLine("Recovered");
    }

    // Valid: if TryCalculate throws, process aborts immediately
    TryCalculate(5);
}

Propagate() throws
{
    TryCalculate(-1);
}
```

## Abort and Panic Mechanics

* MUST terminate the process immediately via Abort or Fatal Panic mechanics when:
  * An uncaught error escapes a non-throwing method, a `where` type guard predicate, a `Dispose(ref @)` method, a `rollback` block, a non-throwing `concurrent` block, or `Program.Run`.
  * An explicit fail-fast operator (`try!`, `must!`, `match!`, `as!`, postfix `!`, or weak interface force-call `Member!`) fails.
  * A thread attempts reentrant acquisition of a spinlock-backed `astatic` variable or field.
* MUST trigger an immediate process abort without stack unwinding when executing in Release mode.
* MUST transition an abort condition into a Fatal Panic when executing in Debug mode.
* MUST unwind the call stack to generate diagnostic trace data during a Fatal Panic in Debug mode.
* MUST bypass all user-defined `catch` handlers during Abort or Fatal Panic mechanics.
* MUST terminate the process immediately upon completion of unwinding in Debug mode.

## Throwing and Catching

* MAY implement `~Disposable` on a `Data` type implementing `~Error`.
* MUST NOT allocate heap memory for the error propagation mechanism during the execution of a `throw` statement.
* MUST NOT allow `Data` types implementing `~Error` to exceed a byte size of `112` bytes or an alignment of `16` bytes.
* MUST write the `(BaseType, ~Error)` `vtable_ptr`, the 64-bit `type_id` of the underlying base `Data` type, and the evaluated error `Data` value bitwise into the pre-allocated 128-byte Thread-Local Storage (TLS) error slot upon executing `throw Expression;`.
* MUST record the `vtable_ptr` and `type_id` of the enclosing base `Data` type when the thrown expression is statically typed as a variant subtype (`Data::Variant`) or a guarded type (`Data^Guard`).
* MUST transfer ownership of the Owning Type (if `~Disposable`) or any nested Owning Type and `T weak` fields (respecting the active discriminant tag for tagged `Data` types) into the TLS error slot without modifying `strongCount` or `weakCount` when the thrown expression is an rvalue.
* MUST atomically increment `strongCount` (for an Owning Type error or nested Owning Type fields) and `weakCount` (for nested `T weak` fields, respecting the active discriminant tag for tagged `Data` types) when the thrown expression is an lvalue.
* MUST propagate thrown errors via deterministic frame-return propagation without table-driven stack unwinding:
  1. Upon executing `throw` (or raising a built-in runtime error such as `DivisionByZeroError`, `OverflowError`, `OutOfMemoryError`, `CastError`, or `InvariantError`), the active frame MUST populate the 128-byte TLS error slot, set the active error slot indicator (register or pointer to the TLS slot), execute error-path cleanup for the exited scopes, and either transfer control to an enclosing `try` / `catch` construct or return to the caller with an uninitialized (`noinit`) return value payload.
  2. Immediately following any potentially throwing invocation or operation, the caller MUST inspect the active error slot indicator: if clear (null), normal execution continues; if set, the caller MUST ignore the `noinit` return payload and either intercept the error (`catch` or `try?`), trigger Abort or Fatal Panic mechanics (in a non-throwing context or `try!`), or execute its own error-path cleanup and return `noinit` to its caller.
* MUST protect the active propagating error during error-path cleanup whenever exiting a scope executes any activated `undo` statement, evaluates an error-return `where` invariant check, releases a `catch` construct's frame-local error slot, or destroys a local value whose cleanup executes user-defined `Dispose` code:
  1. Allocate a dedicated 128-byte frame-local cleanup save slot per lexical cleanup nesting level, isolated from all `try` / `catch` frame-local error slots.
  2. Move the 128-byte error slot from the TLS error slot into the frame-local cleanup save slot and clear the active error slot indicator and TLS error slot (`vtable_ptr = 0x0`) once prior to executing the scope's error-path cleanup.
  3. Execute all activated `undo` statements, local ARC / `~Disposable` cleanups, error-return invariant checks, and enclosing `catch` slot cleanups in reverse declaration (LIFO) order while the TLS error slot remains clear.
  4. Move the 128-byte error slot from the frame-local cleanup save slot back into the TLS error slot and restore the active error slot indicator upon completing error-path cleanup, prior to transferring control to an enclosing `try` / `catch` construct or returning to the caller.
* Implementations MAY elide allocation of and moves through the frame-local cleanup save slot on an error-cleanup path when statically guaranteed that neither an `undo` statement, an error-return invariant check, nor any cleanup operation on that path accesses the TLS error slot.
* MUST allocate a dedicated, isolated 128-byte frame-local error slot for each lexical `try` / `catch` nesting level within a method frame.
* MUST restrict the filter type of a typed `catch` arm (`catch [ref] e: Type`) to:
  * `~Error` or `~Error^Guard`, or
  * A concrete `Data` type implementing `~Error`, optionally narrowed to a tagged variant subtype (`Data::Variant`) and/or governed by a type guard (`^Guard`).
* MUST reject at compile time any `catch` arm whose filter type is an interface other than `~Error` (or `~Error^Guard`), a concrete `Data` type not implementing `~Error`, a variant subtype of an untagged union (`tag none`), or any non-`Data` type.
* MUST move the 128-byte error slot (metadata header and payload) from the TLS error slot into the `try` / `catch` construct's dedicated 128-byte frame-local error slot and clear the active error slot indicator and TLS error slot (`vtable_ptr = 0x0`) upon entering `catch` dispatch, prior to evaluating any `catch` arm filter or type guard `where` clause.
* MUST evaluate `catch` arms sequentially from top to bottom against the frame-local error slot and execute the first matching arm:
  * `catch [ref] e: ConcreteType` matches when the frame-local slot's `type_id` equals the `type_id` of `ConcreteType`.
  * `catch [ref] e: ConcreteType::Variant` matches when the frame-local slot's `type_id` equals the `type_id` of `ConcreteType` and the active discriminant tag in the slot's payload equals `tagof ConcreteType::Variant`.
  * `catch [ref] e: ~Error` matches any active error.
  * A parameterless `catch` arm matches any active error.
* MUST move the 128-byte error slot from the frame-local error slot back into the TLS error slot, restore the active error slot indicator, and continue error propagation (or trigger Abort or Fatal Panic mechanics in a non-throwing context) if no `catch` arm matches.
* Implementations MAY defer moving the error slot into the frame-local error slot until a `catch` arm matches when no `catch` arm of that `try` / `catch` construct evaluates a type guard `where` clause.
* MUST treat a non-`ref` `catch` binding (`catch e: Type`) as an immutable read-only view of the `catch` construct's frame-local error slot for both concrete `Data` types and `~Error`:
  * MUST NOT reassign `e`, mutate its fields, switch its active variant via `=::`, pass `e` or its fields as `ref` arguments, or invoke mutating (`ref @`) methods on `e`.
* MAY declare a mutable `catch` binding by prefixing the identifier with `ref` (`catch ref e: Type`):
  * When `Type` is a concrete `Data` type (`Data`, `Data::Variant`, `Data^Guard`, or `Data::Variant^Guard`), `e` MUST bind directly as a mutable in-place reference to the frame-local error slot's payload, subject to standard `Data::Variant` and `TypeGuards` mutability rules.
  * When `Type` is `~Error` or `~Error^Guard`, `e` MUST bind as a mutable 2-word fat pointer `(data_ptr, vtable_ptr)` referencing the frame-local error slot's payload, permitting both `@` and `ref @` object-safe method invocations in-place and passing `ref e` to `ref ~Error` parameters, while prohibiting direct reassignment (`e = ...`).
  * In-place mutations performed through a `catch ref e` binding MUST modify the frame-local error slot directly and persist if the error is rethrown via a parameterless `throw;` statement (or `=> throw` in a `try ... catch` expression).
* MAY omit the variable identifier and type in a `catch` arm to define a catch-all arm.
* MAY execute a parameterless `throw;` statement (or `=> throw` in a `try ... catch` expression) within a `catch` arm to rethrow the active error.
* MUST move the 128-byte error slot (metadata header and payload) from the innermost enclosing `catch` construct's frame-local error slot back into the TLS error slot and clear the frame-local error slot (`vtable_ptr = 0x0`) when executing a parameterless `throw;` statement or `=> throw`.
* MUST NOT execute a parameterless `throw;` statement outside of a `catch` arm.
* MUST invoke the `dispose_fn` recorded in word `2` of the frame-local error slot's `vtable_ptr` (when non-null) on the slot's payload and clear the frame-local error slot (`vtable_ptr = 0x0`) upon exiting the `catch` arm via normal completion, value yield, control-flow transfer (`return`, `break`, `continue`), or propagation of a newly thrown error (`throw Expression;` or an uncaught potentially throwing operation), omitting cleanup only when the intercepted error is rethrown via parameterless `throw;` or `=> throw`.
* MUST dispose an active error intercepted by `try?` as follows:
  1. Read `dispose_fn` from word `2` of the TLS error slot's `vtable_ptr`.
  2. If `dispose_fn` is `0x0`, clear the active error slot indicator and TLS error slot (`vtable_ptr = 0x0`).
  3. If `dispose_fn` is non-null (`!= 0x0`), move the 128-byte error slot from the TLS error slot into a frame-local temporary error slot, clear the active error slot indicator and TLS error slot (`vtable_ptr = 0x0`), and invoke `dispose_fn` on the temporary slot's payload.

```scatty
AppError : ~Error
{
    ::Timeout(DurationMs: Int);
    ::Disconnected;
}

HttpError : ~Error
{
    Code: Int;
    Message: String;

    AppendContext(ref @, suffix: String)
    {
        @.Message = "\(@.Message): \(suffix)";
    }
}

DbResourceError : ~Error, ~Disposable
{
    _handle: Int;
    Message: String;

    Dispose(ref @) => OS.Close(@._handle);
}

RunRequest() throws { }

HandleErrors() throws
{
    try
    {
        RunRequest();
    }
    catch e: AppError::Timeout
    {
        // Valid: 'e' is read-only and narrowed to 'AppError::Timeout'
        // 'Message' is omitted
        Console.WriteLine("Timeout after \(e.DurationMs) ms");
    }
    catch ref e: HttpError
    {
        // Valid: 'ref' allows in-place mutation preserved by 'throw;'
        e.AppendContext("gateway");
        throw;
    }
    catch e: ~Error
    {
        // Valid: 'Message?' is a weak contract on '~Error'
        Console.WriteLine(e.Message? ?? "Unknown error");
        throw;
    }
    catch
    {
        Console.WriteLine("Unhandled error");
    }
}

InvalidCatches() throws
{
    try
    {
        RunRequest();
    }
    catch e: HttpError
    {
        e.Code = 500; // ERROR: non-'ref' catch binding is read-only
    }
    catch ref e: ~Error
    {
        e = HttpError { Code: 500, Message: "Replaced" }; // ERROR: cannot reassign '~Error'
    }
    catch e: ~Error
    {
        Console.WriteLine(e.Message); // ERROR: unverified weak property 'Message?'
    }
    catch e: ~Formattable // ERROR: only '~Error' is permitted as an interface catch filter
    {
        Console.WriteLine(e.ToString());
    }
}
```

## `try ... catch` Expressions

* MAY evaluate a `try ... catch` expression (`try Expr catch_expr_arm { catch_expr_arm }`) to yield a value of type `T`.
* MUST require the tried expression (`Expr`) of a `try ... catch` expression to be a potentially throwing expression evaluating to a non-void type `T`.
* MUST parse an unparenthesized `{` token immediately following `try` as the block body of a `try ... catch` statement, never as an unqualified collection initializer of a `try ... catch` expression.
* MUST permit each `catch` arm of a `try ... catch` expression to take one of two forms:
  * A value-yielding arrow arm (`catch [filter] => FallbackExpr`), where `FallbackExpr` MUST evaluate to the exact same type `T` as the tried expression (with `T` serving as the expected type context for `FallbackExpr`).
  * A diverging arm (`catch [filter] => (throw [Expr] | return [Expr] | break | continue)` or `catch [filter] { ... }`), where a block arm MUST unconditionally exit the enclosing scope via `return`, `throw`, `break`, or `continue`.
* MUST scope any `return` statement inside a `try ... catch` expression arm to the immediately enclosing method, anonymous function, or `spawn` block, never to the `try ... catch` expression itself.
* MUST require a nested `try ... catch` expression appearing inside a `try` operand or a `catch =>` arm to be enclosed in parentheses `(...)`.
* MUST apply the exact same frame-local error slot dispatch, filtering, mutability (`ref`), rethrow (`throw` / `throw;`), and `dispose_fn` cleanup rules to `try ... catch` expressions as to `try ... catch` statements.
* In statement position, an unparenthesized `"{"` or `"=>"` token immediately following `"try"` MUST be parsed as `try_catch_stmt`, never as `try_catch_expr`.
* Within `catch_expr_body`, a `"throw"` or `"return"` token immediately followed by `"catch"`, `";"`, `","`, `")"`, `"]"`, or `"}"` MUST be parsed without an operand expression.

```scatty
ParseInt(val: String) throws -> Int
{
    may val == "" => throw Error.New("Empty");
    return 42;
}

ReadConfigValue(raw: String, fallback: Int) throws -> Int
{
    // Inline expression fallback with multiple typed catch arms
    parsed := try ParseInt(raw)
        catch e: OutOfMemoryError => throw
        catch e: ~Error => fallback;

    // Diverging block arm ('return' exits 'ReadConfigValue')
    ratio := try 100 / parsed
        catch e: DivisionByZeroError
        {
            Console.WriteLine("Zero divisor");
            return 0;
        };

    // Nested 'try ... catch' expression requires parentheses
    recovered := try ParseInt(raw) catch => (try ParseInt("1") catch => 0);

    return ratio + recovered;
}
```

## `try?` Operator

* MUST apply `try?` exclusively to potentially throwing expressions.
* MUST NOT apply `try?` to non-throwing expressions.
* MUST evaluate `try?` to `T?` (`Option<T>`) when applied to an expression returning a value of type `T`.
* MUST yield `Option::Some` containing the returned value upon successful completion of a value-returning expression.
* MUST intercept and dispose any thrown error (moving out of and clearing the TLS error slot prior to invoking a non-null `dispose_fn`) and yield `Option::None` if a value-returning expression throws.
* MUST evaluate `try?` to `Bool` when applied to an expression returning void.
* MUST yield `true` upon successful completion of a void-returning expression.
* MUST intercept and dispose any thrown error (moving out of and clearing the TLS error slot prior to invoking a non-null `dispose_fn`) and yield `false` if a void-returning expression throws.

```scatty
FetchUser(id: Int) throws -> User
{
    may id < 0 => throw Error.New("Invalid ID");
    return User.New(id);
}

Ping() throws
{
    throw Error.New("Timeout");
}

// 'user' is 'User?' (Option<User>)
user := try? FetchUser(10);

// 'quotient' is 'Int?' (Option::None if divisor is 0)
quotient := try? (total / count);

// 'isAlive' is 'Bool' (true if succeeded, false if threw)
isAlive := try? Ping();
```

## `try!` Operator and Block

* MUST apply `try!` to potentially throwing expressions or to blocks that contain at least one potentially throwing operation.
* MUST evaluate `try! Expression` to the exact return type of the underlying expression.
* MUST trigger runtime Abort or Fatal Panic mechanics immediately (bypassing any enclosing `catch` handlers) if the expression evaluated by `try!` throws an error.
* MAY apply `try!` to a statement block (`try! { ... }`).
* MUST evaluate a `try! { ... }` block as void.
* MUST trigger runtime Abort or Fatal Panic mechanics immediately (bypassing any enclosing `catch` handlers) if any statement within a `try! { ... }` block throws an error.

```scatty
ParseInt(val: String) throws -> Int
{
    may val == "" => throw Error.New("Empty String");
    return 0;
}

ValidateState() throws { }
CommitState() throws { }

ProcessData() throws
{
    val := try! ParseInt("123");

    try!
    {
        ValidateState();
        CommitState();
    }
}
```

## Type Guard Filtering in Catch Blocks

* MAY specify a Type Guard on a concrete `Data` error type (`catch [ref] e: Type^Guard`), a tagged variant error subtype (`catch [ref] e: Type::Variant^Guard`), or `~Error` (`catch [ref] e: ~Error^Guard`).
* MUST evaluate the type guard's `where` expression against the active error instance after the base `type_id` (and active discriminant tag, for `Type::Variant^Guard`) matches.
* MUST intercept the error and refine the binding's static type to the guarded type within the `catch` arm if the `where` expression evaluates to `true`.
* MUST bypass the `catch` arm and continue evaluating subsequent arms or propagate to the caller if the `where` expression evaluates to `false`.

```scatty
HttpError : ~Error
{
    Code: Int;
    Message: String;
}

^ClientError = HttpError where @.Code >= 400 and @.Code < 500;
^ServerError = HttpError where @.Code >= 500 and @.Code < 600;
^EmptyMessage = ~Error where @ is .Message and @.Message == "";

ProcessRequest() throws
{
    throw HttpError { Code: 404, Message: "Not Found" };
}

HandleRequest() throws
{
    try
    {
        ProcessRequest();
    }
    catch e: HttpError^ClientError
    {
        Console.WriteLine("Client fault: \(e.Code)");
    }
    catch e: HttpError^ServerError
    {
        Console.WriteLine("Server fault: \(e.Code)");
    }
    catch e: ~Error^EmptyMessage
    {
        Console.WriteLine("Error with empty message");
    }
    catch e: ~Error
    {
        throw;
    }
}
```

## `undo` Statement

* MAY declare an `undo` statement (`undo => Statement;` or `undo { ... }`) within a block.
* MUST reject an `undo` statement at compile time if no potentially throwing operation follows it within the same lexical block.
* MUST activate an `undo` statement only after execution reaches its declaration point.
* MUST execute activated `undo` statements if and only if execution exits the enclosing lexical block via an uncaught error.
* MUST NOT execute `undo` statements when execution exits the enclosing block normally or via `return`, `break`, `continue`, `fall`, or a non-throwing `must` gate.
* MUST execute multiple activated `undo` statements within the same lexical block in reverse order of declaration (Last-In, First-Out).
* MUST execute activated `undo` statements and local variable ARC / `~Disposable` cleanup in reverse declaration order, executing each `undo` statement prior to destroying any local variables declared above it in the same block.
* MUST evaluate and mutate local variables, `ref` parameters, and `ref @` receivers within an `undo` body directly against the active stack frame's state during error propagation prior to frame return.
* MUST keep the active propagating error preserved in the frame-local cleanup save slot—with the active error slot indicator and TLS error slot cleared—throughout the execution of all activated `undo` statements and interleaved local ARC / `~Disposable` cleanups, restoring the error into the TLS error slot upon completion of that scope's error-path cleanup.
* MUST NOT invoke a `throws` method directly within an `undo` body unless wrapped in `try?`, `try!`, or a `try` / `catch` construct that handles all errors within the `undo` scope; any uncaught error raised by a runtime operation inside an `undo` body MUST immediately trigger Abort or Fatal Panic mechanics.
* MUST NOT execute a `throw`, `return`, `break`, `continue`, or `fall` statement that exits an `undo` body.

```scatty
Account
{
    Balance: Int;
    Users: [Int: User];
    Index: [String: Int];

    RegisterAndCharge(ref @, user: User, fee: Int) throws
    {
        @.Users[user.Id] = user;
        undo => @.Users.Remove(user.Id);

        @.Index[user.Email] = user.Id;
        undo
        {
            @.Index.Remove(user.Email);
            try? AuditLog.WriteRevert(user.Id);
        }

        @.Balance += fee;
        undo => @.Balance -= fee;

        // If 'SendConfirmation' throws, active 'undo' blocks execute in LIFO order:
        // 1. '@.Balance -= fee'
        // 2. '@.Index.Remove(user.Email)' and 'try? AuditLog.WriteRevert(user.Id)'
        // 3. '@.Users.Remove(user.Id)'
        SendConfirmation(user.Email);
    }
}
```
