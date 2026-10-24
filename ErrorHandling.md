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
    if value < 0 => throw Exception.New("Less than zero!");
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
* MUST copy the evaluated exception `Data` value bitwise into the pre-allocated Thread-Local Storage (TLS) exception buffer.
* MUST NOT allow `Data` types implementing `~Exception` to exceed the fixed 128-byte size limit of the TLS exception buffer.
* MUST unwind the call stack until an execution frame containing a matching `catch` arm is reached.
* MUST execute the first matching `catch` arm for an intercepted exception.
* MUST copy the exception data from the TLS exception buffer into the local variable declared by the matching `catch` arm and clear the slot.
* MAY omit the variable identifier and type in a `catch` arm to define a catch-all block.
* MAY execute a parameterless `throw;` statement within a `catch` or `rollback` block to rethrow the active exception.
* MUST NOT execute a parameterless `throw;` statement outside of a `catch` or `rollback` block.

```C#
Process(value: Int) throws
{
    if value < 0 => throw Exception.New("Negative value");
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
* MUST evaluate `try?` to `Result<T>` when applied to an expression returning a value of type `T`.
* MUST yield `Result::Ok` containing the returned value upon successful completion of the expression.
* MUST yield `Result::Error` containing the intercepted exception instance if the expression throws.
* MUST evaluate `try?` to `Option<~Exception>` when applied to an expression returning void.
* MUST yield `Option::None` upon successful completion of a void-returning expression.
* MUST yield `Option::Some` containing the intercepted exception instance if the void-returning expression throws.

```C#
FetchUser(id: Int) throws -> User 
{
    if id < 0 => throw Exception.New("Invalid ID");
    return User.New(id);
}

Ping() throws 
{
    throw Exception.New("Timeout");
}

// result is Result<User>
result := try? FetchUser(10);

// maybeError is ~Exception?
maybeError := try? Ping();
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
    if val == "" => throw Exception.New("Empty String");
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
* MUST evaluate the type guard's `where` expression against the active exception instance when the base type matches.
* MUST intercept the exception and refine the local variable's type within the `catch` block if the `where` expression evaluates to `true`.
* MUST bypass the `catch` arm and continue evaluating subsequent arms or propagate to the caller if the `where` expression evaluates to `false`.

```C#
HttpException : ~Exception 
{
    Code: Int;
    Message: String;
}

^ClientError = HttpException where @.Code >= 400 and @.Code < 500;
^ServerError = HttpException where @.Code >= 500 and @.Code < 600;

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
