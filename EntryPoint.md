# Program Entry Point

An executable Coggy package MUST declare a public `Data` type named `Program` containing a static method named `Run`. This method serves as the singular entry point for the operating system process.

* **Static Context**: The `Run` method MUST NOT declare an instance receiver (`@`).
* **Location**: The `Program` type MUST be declared at the top level of the package.

The compiler recognizes and binds to the `Run` method if it matches one of the following permutations of parameters and return types.

## Command-Line Arguments

* **Parameterless**: `Run()` - Ignores command-line arguments.
* **With Arguments**: `Run(args: String[])` - The runtime intrinsically populates the array with the command-line arguments provided by the operating system.

## Exit Codes

* **Implicit Success (Void)**: If the return type is omitted, the runtime intrinsically returns an exit code of `0` (Success) to the operating system upon normal termination.
* **Explicit Code (`Int`)**: If the method declares `-> Int`, the developer MUST return an explicit exit code.

## Top-Level Error Handling

The `Run` method MAY declare the `throws` modifier to allow domain exceptions to bubble up to the process boundary.

* **Unhandled Escaping**: If a `throws` entry point yields an unhandled exception, the runtime intercepts it and intrinsically triggers the standard Abort and Panic Mechanics (Fatal Panic in Debug, Immediate Abort in Release).
* **Fault Exit Code**: In the event of an Abort or Panic, the runtime returns a non-zero exit code to the operating system.

```C#
// Minimal Entry Point
Program 
{
    // Implicitly returns 0 on success
    Run() 
    {
        Console.WriteLine("Hello, Coggy!");
    }
}

// Full Entry Point
{
    // Accepts OS arguments, explicitly returns an exit code, and propagates errors
    Run(args: String[]) throws -> Int 
    {
        if args.Length == 0 
        {
            Console.WriteLine("Usage: app <config_path>");
            return 1; // Error exit code
        }

        // If 'Load' fails, the program panics/aborts immediately
        config := try! Config.Load(args[0]);
        
        Engine.Start(config);
        
        return 0; // Success exit code
    }
}
```
