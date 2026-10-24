# Type Extensions

* MUST declare a type extension block using the prefix `+` operator applied to the target type identifier (`+TargetType`).
* MAY extend any `Data` type, including primitive types.
* MAY declare instance methods, static methods, properties, and refinement types within an extension block.
* MUST NOT declare structural state fields within an extension block.
* MUST NOT declare members in an extension block that shadow or collide with members declared in the target type's primary definition.
* MUST NOT declare members in an extension block that shadow or collide with members declared in another extension block for the same target type.
* MAY declare new overloads of an existing method name within an extension block provided their parameter signatures are distinct under Method Overloading rules.
* MUST NOT declare members in an extension block whose signatures collide with or shadow members declared in the target type's primary definition or in another extension block for the same target type.

## Encapsulation and Access Rights

* MUST permit an extension block to access private and internal members of the target type if the extension is declared within the same package as the target type.
* MUST restrict an extension block's access to public members of the target type if the target type is imported from an external package.

```C#
MyData 
{
    _count: Int;
    Increment(ref @) => @._count = @._count + 1;
}

+MyData 
{
    Count: Int => @._count;

    ^ValidData = MyData where @.Count > 0;

    PrintCount(@) => Console.WriteLine(@.Count);
}

+String 
{
    ^Email = String where @.Contains("@");

    Empty() -> String => "";
}

data := MyData { 1 };
validData := data as MyData^ValidData;

email := "test@example.com" as String^Email;
```
