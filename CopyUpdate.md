# Copy-Update

```EBNF
CopyUpdateExpr  ::= Expression "with" CopyUpdateBody
CopyUpdateBody  ::= "{" InitializerList "}"
InitializerList ::= FieldAssignment ("," FieldAssignment)* ","?
FieldAssignment ::= Identifier ":" Expression
```

* MUST perform a structural bitwise copy and apply mutations for Trivial Types.
* MUST move the target expression, reuse allocated resources in-place, and yield the updated instance for Owning Types.
* MUST NOT access the original identifier after a `with` expression consumes an Owning Type.
* MUST overwrite corresponding fields in the copy with explicitly declared assignments in the `CopyUpdateBody`.
* MUST resolve the assigned expression to the exact type of the target field.
* MUST retain the original instance's value for any field omitted from the `CopyUpdateBody`.
* MUST NOT update restricted (private or internal) fields from outside their permitted lexical scope.
* MUST update physical fields only.
* MUST NOT directly update transparently forwarded members of an Inline Field.
* MUST assign a new instance to the root identifier of an Inline Field to mutate its state via `with`.

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

testConfig := defaultConfig with 
{ 
    Port: 9000, 
    Timeout: 5000 
};
```
