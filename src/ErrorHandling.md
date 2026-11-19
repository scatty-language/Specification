# Error Handling

* MUST require types thrown as exceptions to be `Data` types implementing the `~Exception` standard interface.
* MUST require types implementing `~Exception` to provide a `Message: String` property.

## The `throws` Modifier

* MUST classify methods lacking the `throws` modifier as non-throwing.
* MUST NOT execute a `throw Expression;` statement within a non-throwing method.
* MAY invoke a `throws` method within a non-throwing method without a `try` construct.
* MUST immediately terminate the process via Abort or Panic mechanics if an invoked method throws an exception within a non-throwing method.
* MAY execute `throw Expression;` statements within methods declared with the `throws` modifier.
* MUST propagate unhandled exceptions to the calling scope in methods declared with the `throws` modifier.

```scatty
TryCalculate(value: Int) throws 
{
    may value < 0 => throw Exception.New("Less than zero!");
}

Calculate() 
{
    // Valid: Handled via try/catch
    try 
    {
        TryCalculate(-1);
    } 
    catch 
    {
        Console.WriteLine("Recovered");
    }

    // Valid: If TryCalculate throws, process aborts immediately
    TryCalculate(5); 
}

Some() throws 
{
    TryCalculate(-1);
}
```

## Abort and Panic Mechanics

* MUST terminate the process if an exception escapes an uncaught scope or is raised inside a non-throwing method.
* MUST trigger an immediate process abort without stack unwinding when executing in Release mode.
* MUST transition an unhandled exception into a Fatal Panic when executing in Debug mode.
* MUST unwind the call stack to generate diagnostic trace data during a Fatal Panic in Debug mode.
* MUST bypass all user-defined `catch` blocks during a Fatal Panic.
* MUST terminate the process immediately upon completion of unwinding in Debug mode.

## Throwing and Catching

* MAY implement `~Disposable` on a `Data` type implementing `~Exception`.
* MUST NOT allocate heap memory for the exception propagation mechanism during the execution of a `throw` statement.
* MUST NOT allow `Data` types implementing `~Exception` to exceed a byte size of `112` bytes or an alignment of `16` bytes.
* MUST write the `(BaseType, ~Exception)` `vtable_ptr`, the 64-bit `type_id` of the underlying base `Data` type, and the evaluated exception `Data` value bitwise into the pre-allocated 128-byte Thread-Local Storage (TLS) exception slot upon executing `throw Expression;`.
* MUST record the `vtable_ptr` and `type_id` of the enclosing base `Data` type when the thrown expression is statically typed as a variant subtype (`Data::Variant`) or a guarded type (`Data^Guard`).
* MUST transfer ownership of the Owning Type (if `~Disposable`) or any nested Owning Type and `T weak` fields (respecting the active discriminant tag for tagged `Data` types) into the TLS exception slot without modifying `strongCount` or `weakCount` when the thrown expression is an rvalue.
* MUST atomically increment `strongCount` (for an Owning Type exception or nested Owning Type fields) and `weakCount` (for nested `T weak` fields, respecting the active discriminant tag for tagged `Data` types) when the thrown expression is an lvalue.
* MUST propagate thrown exceptions via deterministic frame-return propagation without table-driven stack unwinding:
  1. Upon executing `throw`, the active frame MUST populate the 128-byte TLS exception slot, set the active exception slot indicator (register or pointer to the TLS slot), execute any activated frame-local `undo` statements and local ARC / `~Disposable` cleanup, and return to the caller with an uninitialized (`noinit`) return value payload.
  2. Immediately following any `throws` invocation, the caller MUST inspect the active exception slot indicator: if clear (null), normal execution continues; if set, the caller MUST ignore the `noinit` return payload and either intercept the exception (`catch` or `try?`), trigger Abort/Panic (in a non-throwing context or `try!`), or execute its own activated `undo` statements and local ARC / `~Disposable` cleanup and return `noinit` to its caller.
* MUST allocate a dedicated, isolated 128-byte frame-local exception slot for each lexical `try` / `catch` nesting level within a method frame.
* MUST restrict the filter type of a typed `catch` arm (`catch [ref] e: Type`) to:
  * `~Exception` or `~Exception^Guard`, or
  * A concrete `Data` type implementing `~Exception`, optionally narrowed to a tagged variant subtype (`Data::Variant`) and/or governed by a type guard (`^Guard`).
* MUST reject at compile time any `catch` arm whose filter type is an interface other than `~Exception` (or `~Exception^Guard`), a concrete `Data` type not implementing `~Exception`, a variant subtype of an untagged union (`tag none`), or any non-`Data` type.
* MUST evaluate `catch` arms sequentially from top to bottom and execute the first matching arm:
  * `catch [ref] e: ConcreteType` matches when the active exception's `type_id` equals the `type_id` of `ConcreteType`.
  * `catch [ref] e: ConcreteType::Variant` matches when the active exception's `type_id` equals the `type_id` of `ConcreteType` and the active discriminant tag in the exception payload equals `tagof ConcreteType::Variant`.
  * `catch [ref] e: ~Exception` matches any active exception.
  * A parameterless `catch` arm matches any active exception.
* MUST move the 128-byte exception slot (metadata header and payload) from the TLS exception slot into the `try` / `catch` construct's dedicated 128-byte frame-local exception slot and clear the active exception slot indicator and TLS exception slot upon entering a matching `catch` arm.
* MUST treat a non-`ref` `catch` binding (`catch e: Type`) as an immutable read-only view of the `catch` construct's frame-local exception slot for both concrete `Data` types and `~Exception`:
  * MUST NOT reassign `e`, mutate its fields, switch its active variant via `=::`, pass `e` or its fields as `ref` arguments, or invoke mutating (`ref @`) methods on `e`.
* MAY declare a mutable `catch` binding by prefixing the identifier with `ref` (`catch ref e: Type`):
  * When `Type` is a concrete `Data` type (`Data`, `Data::Variant`, `Data^Guard`, or `Data::Variant^Guard`), `e` MUST bind directly as a mutable in-place reference to the frame-local exception slot's payload, subject to standard `Data::Variant` and `TypeGuards` mutability rules.
  * When `Type` is `~Exception` or `~Exception^Guard`, `e` MUST bind as a mutable 2-word fat pointer `(data_ptr, vtable_ptr)` referencing the frame-local exception slot's payload, permitting both `@` and `ref @` object-safe method invocations in-place and passing `ref e` to `ref ~Exception` parameters, while prohibiting direct reassignment (`e = ...`).
  * In-place mutations performed through a `catch ref e` binding MUST modify the frame-local exception slot directly and persist if the exception is rethrown via a parameterless `throw;` statement.
* MAY omit the variable identifier and type in a `catch` arm to define a catch-all block.
* MAY execute a parameterless `throw;` statement within a `catch` block to rethrow the active exception.
* MUST move the 128-byte exception slot (metadata header and payload) from the innermost enclosing `catch` construct's frame-local exception slot back into the TLS exception slot when executing a parameterless `throw;` statement.
* MUST NOT execute a parameterless `throw;` statement outside of a `catch` block.
* MUST invoke the `dispose_fn` recorded in word `2` of the frame-local exception slot's `vtable_ptr` (when non-null) on the slot's payload upon exiting the `catch` block via normal completion, control-flow transfer (`return`, `break`, `continue`), or propagation of a newly thrown exception (`throw Expression;` or an uncaught `throws` invocation), omitting cleanup only when the intercepted exception is rethrown via a parameterless `throw;` statement.
* MUST invoke the `dispose_fn` recorded in word `2` of the TLS exception slot's `vtable_ptr` (when non-null) on the slot's payload and clear the TLS exception slot when an active exception is intercepted by `try?`.

```scatty
AppError : ~Exception
{
    ::Timeout(DurationMs: Int);
    ::Disconnected;

    Message: String;
}

HttpException : ~Exception
{
    Code: Int;
    Message: String;

    AppendContext(ref @, suffix: String)
    {
        @.Message = "\(@.Message): \(suffix)";
    }
}

DbResourceException : ~Exception, ~Disposable
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
        Console.WriteLine("Timeout after \(e.DurationMs) ms: \(e.Message)");
    }
    catch ref e: HttpException
    {
        // Valid: 'ref' allows in-place mutation preserved by 'throw;'
        e.AppendContext("gateway");
        throw;
    }
    catch e: ~Exception
    {
        Console.WriteLine(e.Message);
        throw;
    }
    catch
    {
        Console.WriteLine("Unknown error");
    }
}

InvalidCatches() throws
{
    try
    {
        RunRequest();
    }
    catch e: HttpException
    {
        e.Code = 500; // Invalid: non-'ref' catch binding is read-only — MUST be rejected
    }
    catch ref e: ~Exception
    {
        e = HttpException { Code: 500, Message: "Replaced" }; // Invalid: cannot reassign '~Exception' — MUST be rejected
    }
    catch e: ~Formattable // Invalid: only '~Exception' is permitted as an interface catch filter — MUST be rejected
    {
        Console.WriteLine(e.ToString());
    }
}
```

## `try?` Operator

* MUST apply `try?` to expressions that invoke a `throws` method.
* MUST NOT apply `try?` to non-throwing expressions.
* MUST evaluate `try?` to `T?` (`Option<T>`) when applied to an expression returning a value of type `T`.
* MUST yield `Option::Some` containing the returned value upon successful completion of a value-returning expression.
* MUST intercept any thrown exception, clear the TLS exception buffer, and yield `Option::None` if a value-returning expression throws.
* MUST evaluate `try?` to `Bool` when applied to an expression returning void.
* MUST yield `true` upon successful completion of a void-returning expression.
* MUST intercept any thrown exception, clear the TLS exception buffer, and yield `false` if a void-returning expression throws.

```scatty
FetchUser(id: Int) throws -> User 
{
    may id < 0 => throw Exception.New("Invalid ID");
    return User.New(id);
}

Ping() throws 
{
    throw Exception.New("Timeout");
}

// 'user' is 'User?' (Option<User>)
user := try? FetchUser(10);

// 'isAlive' is 'Bool' (true if succeeded, false if threw)
isAlive := try? Ping();
```

## `try!` Operator and Block

* MUST apply `try!` to expressions or blocks that contain at least one invocation of a `throws` method.
* MUST evaluate `try! Expression` to the exact return type of the underlying expression.
* MUST trigger runtime Abort or Panic mechanics immediately if the expression evaluated by `try!` throws an exception.
* MAY apply `try!` to a statement block (`try! { ... }`).
* MUST evaluate a `try! { ... }` block as void.
* MAY invoke `throws` methods directly without individual `try` qualifiers within a `try! { ... }` block.
* MUST trigger runtime Abort or Panic mechanics immediately if any statement within a `try! { ... }` block throws an exception.

```scatty
ParseInt(val: String) throws -> Int 
{
    may val == "" => throw Exception.New("Empty String");
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

* MAY specify a Type Guard on a concrete `Data` exception type (`catch [ref] e: Type^Guard`), a tagged variant exception subtype (`catch [ref] e: Type::Variant^Guard`), or `~Exception` (`catch [ref] e: ~Exception^Guard`).
* MUST evaluate the type guard's `where` expression against the active exception instance after the base `type_id` (and active discriminant tag, for `Type::Variant^Guard`) matches.
* MUST intercept the exception and refine the binding's static type to the guarded type within the `catch` block if the `where` expression evaluates to `true`.
* MUST bypass the `catch` arm and continue evaluating subsequent arms or propagate to the caller if the `where` expression evaluates to `false`.

```scatty
HttpException : ~Exception 
{
    Code: Int;
    Message: String;
}

^ClientError = HttpException where @.Code >= 400 and @.Code < 500;
^ServerError = HttpException where @.Code >= 500 and @.Code < 600;
^EmptyMessage = ~Exception where @.Message == "";

ProcessRequest() throws 
{
    throw HttpException { Code: 404, Message: "Not Found" };
}

HandleRequest() throws 
{
    try 
    {
        ProcessRequest();
    } 
    catch e: HttpException^ClientError 
    {
        Console.WriteLine("Client fault: \(e.Code)");
    }
    catch e: HttpException^ServerError 
    {
        Console.WriteLine("Server fault: \(e.Code)");
    }
    catch e: ~Exception^EmptyMessage
    {
        Console.WriteLine("Exception with empty message");
    }
    catch e: ~Exception 
    {
        throw; 
    }
}
```

## `undo` Statement

* MAY declare an `undo` statement (`undo => Statement;` or `undo { ... }`) within a block.
* MUST reject an `undo` statement at compile time if no `throws` invocation or `throw` statement follows it within the same lexical block.
* MUST activate an `undo` statement only after execution reaches its declaration point.
* MUST execute activated `undo` statements if and only if execution exits the enclosing lexical block via an unhandled exception.
* MUST NOT execute `undo` statements when execution exits the enclosing block normally or via `return`, `break`, `continue`, `fall`, or a non-throwing `must` gate.
* MUST execute multiple activated `undo` statements within the same lexical block in reverse order of declaration (Last-In, First-Out).
* MUST execute activated `undo` statements and local variable ARC / `~Disposable` cleanup in reverse declaration order, executing each `undo` statement prior to destroying any local variables declared above it in the same block.
* MUST evaluate and mutate local variables, `ref` parameters, and `ref @` receivers within an `undo` body directly against the active stack frame's state during exception propagation prior to frame return.
* MUST move the active exception from the TLS exception buffer into a dedicated 128-byte frame-local `undo` exception slot—isolated from any enclosing `catch` construct's frame-local exception slot—prior to executing an `undo` statement, and clear the active exception slot indicator and TLS exception buffer for the duration of the `undo` body.
* MUST move the exception from the dedicated frame-local `undo` exception slot back into the TLS exception buffer and restore the active exception slot indicator upon completion of the `undo` statement.
* MUST NOT invoke a `throws` method within an `undo` body unless wrapped in `try?`, `try!`, or a `try` / `catch` construct that handles all exceptions within the `undo` scope.
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
