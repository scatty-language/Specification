# Type Extensions

* MUST declare a type extension block using the `with` keyword applied to the target type (`TargetType with { ... }`).
* MAY extend any `Data` type, including primitive types.
* MAY declare instance methods, static methods, and properties within an extension block.
* MUST NOT declare structural state fields or type guards within an extension block.
* MUST NOT declare members in an extension block that shadow or collide with members declared in the target type's primary definition.
* MUST NOT declare members in an extension block that shadow or collide with members declared in another extension block for the same target type.
* MAY declare new overloads of an existing method name within an extension block provided their parameter signatures are distinct under Method Overloading rules.

## Encapsulation and Access Rights

* MUST permit an extension block to access private members (`_name`) of the target type if and only if the extension block is declared in the same source file as the target type's primary declaration or in a narrower conditional compilation file sharing the exact same `<BaseName>` within the same directory.
* MUST permit an extension block to access internal members (`Name_`) of the target type if and only if the extension block is declared within the same package as the target type.
* MUST restrict an extension block's access to public members (`Name`) of the target type if the target type is imported from an external package.

```scatty
// File: MyData.cat (PACKAGE: Core)
MyData 
{
    _count: Int;
    Step_: Int = 1;

    Increment(ref @) => @._count = @._count + @.Step_;
}

// Valid: same file as 'MyData' primary declaration; MAY access '_count' and 'Step_'
MyData with 
{
    Count: Int => @._count;

    PrintCount(@) => Console.WriteLine(@.Count);
}

^ValidData = MyData where @.Count > 0;
^Email = String where @.Contains("@");

String with 
{
    Empty() -> String => "";
}

data := MyData { 1 };
validData := data as MyData^ValidData;

email := "test@example.com" as String^Email;

// File: Other.cat (PACKAGE: Core — different <BaseName>)
MyData with 
{
    // Valid: 'Step_' is internal to package 'Core'
    NextStep(@) -> Int => @.Step_;

    // ERROR: '_count' is private to 'MyData.*.cat' files
    LeakCount(@) -> Int => @._count;
}
```
