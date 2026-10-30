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

```C#
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

* MUST NOT allocate heap memory during the execution of a `throw` statement.
* MUST move the evaluated exception `Data` value bitwise into the pre-allocated Thread-Local Storage (TLS) exception buffer, transferring ownership of any nested Owning Type fields (such as `Message: String`) without incrementing their reference counts.
* MUST NOT allow `Data` types implementing `~Exception` to exceed the fixed 128-byte size limit of the TLS exception buffer.
* MUST unwind the call stack until an execution frame containing a matching `catch` arm is reached.
* MUST execute the first matching `catch` arm for an intercepted exception.
* MUST move the exception data from the TLS exception buffer into the local variable declared by the matching `catch` arm and clear the TLS slot.
* MUST release any nested Owning Type fields (decrementing `strongCount` and invoking `Dispose` where applicable) prior to clearing the TLS exception buffer whenever an active exception is suppressed by a parameterless `catch` block, intercepted by `try?`, or overwritten by a new exception.
* MAY omit the variable identifier and type in a `catch` arm to define a catch-all block.
* MAY execute a parameterless `throw;` statement within a `catch` or `rollback` block to rethrow the active exception.
* MUST NOT execute a parameterless `throw;` statement outside of a `catch` or `rollback` block.

```C#
Process(value: Int) throws
{
    may value < 0 => throw Exception.New("Negative value");
}

TryProcess() throws
{
    try 
    {
        Process(-1);
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

```C#
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

```C#
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

* MAY specify a Type Guart as the filter type on a `catch` arm (`catch e: Type^Guard`).
* MUST evaluate the type guard's `must` expression against the active exception instance when the base type matches.
* MUST intercept the exception and refine the local variable's type within the `catch` block if the `must` expression evaluates to `true`.
* MUST bypass the `catch` arm and continue evaluating subsequent arms or propagate to the caller if the `must` expression evaluates to `false`.

```C#
HttpException : ~Exception 
{
    Code: Int;
    Message: String;
}

^ClientError = HttpException must @.Code >= 400 and @.Code < 500;
^ServerError = HttpException must @.Code >= 500 and @.Code < 600;

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
* MUST execute activated `undo` statements and local variable ARC / `~Disposable` cleanup in strict reverse declaration order, executing each `undo` statement prior to destroying any local variables declared above it in the same block.
* MUST evaluate and mutate local variables, `ref` parameters, and `ref @` receivers within an `undo` body directly against the active stack frame's state at the moment of unwinding.
* MUST automatically continue propagating the active exception in the TLS exception buffer after an `undo` statement completes.
* MUST NOT invoke a `throws` method within an `undo` body unless wrapped in `try?`, `try!`, or a `try` / `catch` construct that handles all exceptions within the `undo` scope.
* MUST NOT execute a `throw`, `return`, `break`, `continue`, or `fall` statement that exits an `undo` body.

```C#
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
