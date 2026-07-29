# Copy-Update

The `with` expression allows to create a new instance of a `Data` type by copying an existing instance and mutating specific fields inline.

```EBNF
CopyUpdateExpr  ::= Expression "with" "{" InitializerList "}"
InitializerList ::= FieldAssignment ("," FieldAssignment)* ","?
FieldAssignment ::= Identifier ":" Expression
```

* **Value Semantics**: The expression performs a logically independent structural copy of the target `Expression`.
* **Selective Mutation**: The fields explicitly declared within the `{}` block overwrite the corresponding fields in the new copy. The provided expressions MUST match the field types.
* **Omission Guarantee**: Any field NOT explicitly listed within the `{}` block retains its exact value from the original instance.
* **Visibility Constraints**: The `with` expression respects standard visibility rules. A developer MUST NOT update restricted (private or internal) fields from an external scope.
* **No Implicit Forwarding**: The `with` expression initializes physical fields. Updating transparently forwarded members of an Inline Field directly is FORBIDDEN, the developer MUST assign a new instance to the inline field's root identifier.

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

// Creates a completely independent copy, modifying ONLY 'Port' and 'Timeout'
// 'Host' is retained as "localhost"
testConfig := defaultConfig with 
{ 
    Port: 9000, 
    Timeout: 5000 
};

// ERROR: Cannot update non-existent or inaccessible fields
badConfig := defaultConfig with { UnknownField: 1 };
```
