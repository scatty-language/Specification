# Copy-Update

* MUST perform a structural bitwise copy and apply mutations for Trivial Types.
* MUST preserve the original variable as valid and accessible when evaluating a `with` expression on an Owning Type.
* MUST retain unmodified fields via ARC and assign the newly evaluated expressions to the specified fields in the `CopyUpdateBody`.
* MAY mutate the instance in-place as a compiler optimization if the target expression of `with` is an rvalue or a uniquely referenced temporary (`strongCount == 1`).
* MUST overwrite corresponding fields in the resulting instance with explicitly declared assignments in the `CopyUpdateBody`.
* MUST resolve the assigned expression to the exact type of the target field.
* MUST retain the original instance's value for any field omitted from the `CopyUpdateBody`.
* MUST NOT update restricted (private or internal) fields from outside their permitted lexical scope.
* MUST update physical fields only.
* MUST NOT directly update transparently forwarded members of an Embedded Field.
* MUST assign a new instance to the root identifier of an Embedded Field to mutate its state via `with`.

```C#
ServerConfig 
{
    Host: String;
    Port: Int;
    Timeout: Int;
}

defaultConfig := ServerConfig 
{ 
    Host: "localhost", 
    Port: 8080, 
    Timeout: 3000 
};

// 'defaultConfig' remains valid; 'Host' buffer is shared via ARC
testConfig := defaultConfig with 
{ 
    Port: 9000, 
    Timeout: 5000 
};

Console.WriteLine(defaultConfig.Port); // Output: 8080
Console.WriteLine(testConfig.Port);    // Output: 9000
```
