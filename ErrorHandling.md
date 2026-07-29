# Error Handling

Coggy uses exceptions as the primary mechanism for error handling.

Exceptions MUST be `Data` types implementing the `~Exception` standard interface. The interface dictates that the implementing type MUST provide a `Message` String property.

## `throws` Method Modifier

By default, methods are infallible, with a guarantee that no exceptions will escape their scope.

To allow a method to propagate exceptions to its caller, it MUST explicitly declare the `throws` modifier.

### Non-Throwing Methods

* **Direct Throw Prohibition**: Using the `throw` keyword directly inside a non-`throws` method is FORBIDDEN.
* **Implicit Abort**: A non-`throws` method MAY call a `throws` method without wrapping it in a `try/catch` block. However, if the called method throws an exception, the runtime MUST immediately abort the process.

### Throwing Methods

* **Propagation**: A method marked with `throws` MAY use the `throw` keyword and MAY let exceptions from nested calls bubble up to the caller.

```C#
// Method allows exception propagation
TryCalculate(value: Int) throws 
{
    if value < 0 => throw Exception.New("Less than zero!");
}

// Method is non-throwing by default
Calculate() 
{
    throw Exception.New("Oops!"); // ERROR: Cannot throw directly in a non-throws method

    // Valid
    try 
    {
        TryCalculate(-1);
    } 
    catch 
    {
        Console.WriteLine("Recovered");
    }

    // Valid, if 'TryCalculate(5)' throws, the program ABORTS immediately
    TryCalculate(5); 
}

// Method propagates exceptions from inner calls
Some() throws 
{
    TryCalculate(-1); // Exception bubbles up to the caller of 'Some()'
}
```

## Abort and Panic Mechanics

If an exception is thrown inside a non-`throws` method, or if an exception remains uncaught, the program MUST terminate.

The termination mechanism depends on the active compilation profile.

* **Release Mode (Immediate Abort)**: The compiler MUST emit a process abort. The stack MUST NOT be unwound.
* **Debug Mode (Fatal Panic)**: The runtime MUST transition the unhandled exception into a Fatal Panic.
  * The stack MUST be unwound to generate a stack trace.
  * A Fatal Panic MUST bypass all user-defined `catch` blocks and CANNOT be intercepted.
  * Upon unwinding completion, the process MUST terminate.

## Throwing and Catching

```EBNF
TryCatchStmt  ::= "try" (Block | "=>" Statement) CatchArm+
CatchArm      ::= "catch" [Identifier ":" Type] (Block | "=>" Statement)
ThrowExpr     ::= "throw" [Expression] ";"
```

* **Exception Type Enforcement**: The expression provided to `throw` MUST evaluate to a `Data` type implementing the `~Exception` interface.
* **Rethrowing**: The `throw;` statement WITHOUT an expression is ONLY valid inside a `catch` or `rollback` block. It rethrows the exact same exception instance without copying or reallocating memory.
* **No Fallthrough**: Only the first matching `catch` block executes.

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
    catch e: ~Exception // Catches ANY type implementing the 'Exception' interface
    {
        Console.WriteLine(e.Message);
        throw; // Rethrows to caller
    }
    catch // Catch-all without identifier
    {
        Console.WriteLine("Unknown error");
    }
}
```

### `try?` Operator

For concise inline error handling, the `try?` operator intercepts exceptions from a throwing expression and structurally converts them into returnable data types.

```EBNF
TryOptionExpr ::= "try?" Expression
```

* **Return Type Mapping**: The operator's evaluated type depends on the return signature of the invoked `Expression`.
  * **Value Return**: If the expression returns a type `T`, `try?` evaluates to the standard library `Result<T>` type.
    * Success yields `.Ok` containing the returned value.
    * Thrown exception yields `.Error` containing the exception instance.
  * **Void Return**: If the expression does not return a value, `try?` evaluates to an Option type of `~Exception`.
    * Success yields `None`.
    * Thrown exception yields `Some` containing the exception instance.
* **Throws Requirement**: The `try?` operator MUST ONLY be applied to expressions that invoke a `throws` method. Applying it to a non-throwing expression is FORBIDDEN.

```C#
// Value-returning method
FetchUser(id: Int) throws -> User 
{
    if id < 0 => throw Exception.New("Invalid ID");
    return User.New(id);
}

// Void-returning method
Ping() throws 
{
    throw Exception.New("Timeout");
}

// 'result' is 'Result<User>'
result := try? FetchUser(10);

match result
{
    .Ok(user) => Console.WriteLine(user.Id),
    .Error(e) => Console.WriteLine(e.Message)
}

// 'maybeError' is '~Exception?'
maybeError := try? Ping();

if e := maybeError 
{
    Console.WriteLine(e.Message);
}

// ERROR: Cannot apply 'try?' to a non-throws method
val := try? Math.Abs(-5);
```

### `try!` Operator and Block

The `try!` construct asserts the infallible execution of a block of statements. It explicitly guarantees that the evaluated code will succeed at runtime.

```EBNF
TryForceConstruct ::= "try!" (Expression | Block)
```

* **Expression Mode**:
  * Evaluates to the exact return type of the invoked `Expression`.
  * If the expression throws an exception, the runtime MUST immediately trigger the Abort/Fatal Panic mechanism (bypassing any enclosing `throws` context).
* **Block Mode**:
  * Accepts a block of statements (`{ ... }`).
  * The block MUST be evaluated as `void`.
  * Inside the block, `throws` methods MAY be called without individual `try` modifiers.
  * If ANY statement within the block throws an unhandled exception, the runtime MUST immediately trigger the Abort/Fatal Panic mechanism.
* **Throws Requirement**: The `try!` construct MUST ONLY be applied to expressions or blocks that contain at least one invocation of a `throws` method.

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
    // Returns the unwrapped value or panics
    val := try! ParseInt("123");

    // Asserts that all operations succeed
    // If 'ValidateState' or 'CommitState' throws, the process panics
    try! 
    {
        ValidateState();
        CommitState();
    }
}
```

### Refinement Filtering in Catch Blocks

A `catch` MAY specify a Refinement Type to apply invariant filtering during exception interception.

* **Invariant Evaluation**: When an exception matches the base type of the `catch`, the runtime MUST evaluate the refinement's `where` expression.
* **Conditional Interception**:
  * If the invariant evaluates to `true`, the exception is caught, and its type is implicitly refined within the block.
  * If the invariant evaluates to `false`, the exception bypasses the current `catch` arm and continues evaluating subsequent arms or propagates to the caller.

```C#
HttpException : ~Exception 
{
    Code: Int;
    Message: String; // Implicitly satisfies the ~Exception property contract
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
        Console.WriteLine($"Client fault: {e.Code}");
    }
    catch e: HttpException^ServerError 
    {
        // Skipped: 404 does not match 'ServerError'
        Console.WriteLine($"Server fault: {e.Code}");
    }
    catch e: ~Exception 
    {
        throw; 
    }
}
```
