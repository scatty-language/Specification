# Type Extensions

Extensions allow developers to add new capabilities (methods, properties, and refinements) to existing Data types without modifying their original declaration.

```EBNF
ExtensionDecl ::= "+" Type "{" Member* "}"
```

* **Universal Support**: ANY Data type MAY be extended, including primitive types.
* **Capabilities**: Extensions MAY declare:
  * Instance methods and static methods.
  * Properties.
  * Refinement Types.
* **Prohibitions**:
  * Extensions MUST NOT declare new fields.
  * Extensions MUST NOT declare virtual, abstract, or overriding methods.
  * Extensions MUST NOT declare state transitions.
  * Extensions MUST NOT declare new variants.
  * Extensions MUST NOT declare members that shadows members in declaration or other Extension.

## Encapsulation and Visibility

* **Local Extensions (Same Module)**: If the extension Type is declared in the same module as the original type, it has full access to the type's private fields and methods.
* **External Extensions (Cross-Project)**: If a type is imported from an external compiled project or library, extensions MUST ONLY access its public members.

```C#
MyData 
{
    _count: int;
    Increment(mut @&) => _count++;
}

// Extension block for 'MyData'
+MyData 
{
    // Property extension accessing a private field (valid if in the same module)
    Count: int => @._count;

    // Refinement type scoped to the extension
    ^ValidData = MyData where @.Count > 0;

    // Instance method extension borrowing immutably
    PrintCount(@&) => Console.WriteLine(@.Count);
}

// Extension block for primitive type
+string 
{
    ^Email = string where @.Contains("@");

    Empty() -> string => "";
}

data := MyData {1};
validData := data as MyData::^ValidData;

email := "test@example.com" as string::^Email;
```
