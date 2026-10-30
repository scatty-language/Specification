# Program Entry Point

* MUST declare a public `Data` type named `Program` at the package top level in an executable package.
* MUST declare a public static method named `Run` within the `Program` type as the process entry point.
* MUST NOT declare an instance receiver (`@` or `ref @`) on the `Run` method.
* MUST match the `Run` method signature to one of the permitted parameter and return type combinations.

## Command-Line Arguments

* MAY declare the `Run` method with zero parameters.
* MAY declare the `Run` method with exactly one parameter typed as `[String]`.
* MUST NOT declare parameters with types other than `[String]`.
* MUST NOT declare more than one parameter on the `Run` method.
* MUST populate the `[String]` parameter with command-line arguments provided by the host environment.

## Exit Codes

* MAY omit the return type on the `Run` method.
* MUST return an exit code of `0` to the operating system upon successful termination if the return type is omitted.
* MAY declare the return type of the `Run` method as `-> Int`.
* MUST NOT declare a return type other than `Int` or omitted (void) on the `Run` method.
* MUST return the evaluated `Int` return value to the operating system as the process exit code.

## Top-Level Error Handling

* MAY declare the `Run` method with the `throws` modifier.
* MUST trigger runtime Abort and Panic mechanics if an unhandled exception escapes a `throws` entry point.
* MUST return a non-zero exit code to the operating system if execution terminates via an unhandled exception or process abort.

```C#
// Minimal Entry Point
Program 
{
    Run() 
    {
        Console.WriteLine("Hello, Scatty!");
    }
}

// Full Entry Point
Program
{
    Run(args: [String]) throws -> Int 
    {
        may args.Length == 0 
        {
            Console.WriteLine("Usage: app <config_path>");
            return 1;
        }

        config := try! Config.Load(args[0]);
        Engine.Start(config);
        
        return 0;
    }
}
```
