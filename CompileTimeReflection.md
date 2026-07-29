# Compile-Time Reflection

Coggy provides zero-cost compile-time reflection. Reflection queries are evaluated during compilation and are physically replaced with constant structural literal values in the compiled binary.

```EBNF
ReflectionExpr ::= "meta" "<" Type ">" "." Identifier
```

* **Compile-Time Evaluation**: Any expression starting with `meta<T>` is intrinsically evaluated by the compiler.
* **Standard Library Wrappers**: The compiler maps the reflection data into typed `Data` defined within the `System::Reflection` standard library package.
* **Constant Context**: The evaluated result of a `meta<T>` expression acts as a compile-time constant.

## Reflection Queries

Depending on the `Type` kind passed to `meta<T>`, different queries are available.

* **Fields**: Available for `Data` types. Evaluates to an array of field info.
* **Variants**: Available for Variant Fields. Evaluates to an array of variant info.
* **BaseType**: Available for Refinement Types. Evaluates to a `String` representing the base type's name.
* **Kind**: Available for all types.

```C#
User 
{
    $Id: Int;
    Name: String;
}

// The compiler replaces this with a static array literal
fields: FieldInfo[] = meta<User>.Fields;

for f in fields 
{
    Console.WriteLine(f.Name);
}

^Positive = Int where @ > 0;
base: String = meta<^Positive>.BaseType; // Evaluates to constant "Int"
```
