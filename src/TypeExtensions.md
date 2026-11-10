# Type Extensions

* MUST declare a type extension block using the `with` keyword applied to the target type (`TargetType with { ... }`).
* MAY extend any `Data` type, including primitive types.
* MAY declare instance methods, static methods, and properties within an extension block.
* MUST NOT declare structural state fields or type guards within an extension block.
* MUST NOT declare members in an extension block that shadow or collide with members declared in the target type's primary definition.
* MUST NOT declare members in an extension block that shadow or collide with members declared in another extension block for the same target type.
* MAY declare new overloads of an existing method name within an extension block provided their parameter signatures are distinct under Method Overloading rules.

## Encapsulation and Access Rights

* MUST permit an extension block to access private and internal members of the target type if the extension is declared within the same package as the target type.
* MUST restrict an extension block's access to public members of the target type if the target type is imported from an external package.

```scatty
MyData 
{
    _count: Int;
    Increment(ref @) => @._count = @._count + 1;
}

MyData with 
{
    Count: Int => @._count;

    PrintCount(@) => Console.WriteLine(@.Count);
}

^ValidData = MyData must @.Count > 0;
^Email = String must @.Contains("@");

String with 
{
    Empty() -> String => "";
}

data := MyData { 1 };
validData := data as MyData^ValidData;

email := "test@example.com" as String^Email;
```
