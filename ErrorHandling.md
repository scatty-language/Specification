# Error Handling

Coggy uses exceptions as the primary mechanism for error handling.

Exceptions MUST strictly be Heap-Only Data (`#Data`). To guarantee predictable control flow and systems-level reliability, all methods are strictly non-throwing by default.

## `throws` Function Modifier

By default, functions are infallible, they guarantee no exceptions will escape their scope. To allow a function to propagate exceptions to its caller, it MUST explicitly declare the `throws` modifier.

### Non-Throwing Functions

* **Direct Throw Prohibition**: Using the `throw` keyword directly inside a non-`throws` function is a STRICT compile-time error.
* **Implicit Abort**: A non-`throws` function MAY call a `throws` function without wrapping it in a `try/catch` block. However, if the called function throws an exception, the runtime MUST immediately abort the process.

### Throwing Functions

* **Propagation**: A function marked with `throws` MAY use the `throw` keyword and MAY let exceptions from nested calls bubble up to the caller.

```C#
// Function allows exception propagation
TryCalculate(value: int) throws 
{
    if value < 0 => throw #Exception.New("Less than zero!");
}

// Function is non-throwing by default
Calculate() 
{
    throw #Exception.New("Oops!"); // ERROR: Cannot throw directly in a non-throws function

    // Valid: Handled locally
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

// Function propagates exceptions from inner calls
Some() throws 
{
    TryCalculate(-1); // Exception bubbles up to the caller of 'Some()'
}
```

## Throwing and Catching

```EBNF
TryCatchStmt  ::= "try" (Block | "=>" Statement) CatchArm+
CatchArm      ::= "catch" [Identifier ":" Type] (Block | "=>" Statement)
ThrowStmt     ::= "throw" [Expression] ";"
```

* **Heap-Only Enforcement**: The expression provided to `throw` MUST evaluate to a sharp pointer (`#`) of a Heap-Only Data type.
* **Rethrowing**: The `throw;` statement WITHOUT an expression is ONLY valid inside a `catch` or `rollback` block. It strictly rethrows the exact same exception pointer without reallocating memory.
* **No Fallthrough**: Only the first matching `catch` block executes.

```C#
Process(value: int) throws
{
    if value < 0 => throw #Exception.New("Negative value");
}

TryProcess() throws
{
    try 
    {
        Process(-1);
    } 
    catch e: #Exception 
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

For concise inline error handling, Coggy provides the `try?` operator. It intercepts exceptions from a throwing expression and surfaces them as an Option Pointer.

```EBNF
TryOptionExpr ::= "try?" Expression
```

* **Return Type**: The operator ALWAYS evaluates to `#Exception?`
* **Execution Flow**:
  * If the `Expression` completes successfully, `try?` evaluates to `None`.
  * If the `Expression` throws an exception, the exception is caught, and `try?` evaluates to the exception instance.
* **Value Discard**: If the `Expression` normally returns a value, that value is strictly DISCARDED.
* **Throws Requirement**: The `try?` operator MUST ONLY be applied to expressions that invoke a `throws` method. Applying it to a non-throwing expression is a compilation error.

```C#
// A throwing method
ValidateUser(id: int) throws 
{
    if id < 0 => throw #Exception.New("Invalid user ID");
}

// Inline error capture
maybeError: #Exception? = try? ValidateUser(-1);

// Checking the option pointer
if maybeError != None 
{
    // The pointer is refined to #Exception inside this branch
    Console.WriteLine(maybeError.Message); 
}

// Silent failure suppression
try? ValidateUser(10); // Ignores any thrown exception and discards the result
```
