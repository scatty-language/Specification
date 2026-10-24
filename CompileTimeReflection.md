# Compile-Time Reflection

* MUST initiate compile-time reflection queries using the `meta<TargetType>` syntax.
* MUST evaluate all `meta<TargetType>` expressions during compilation.
* MUST replace `meta<TargetType>` expressions in the compiled output with structural compile-time constant literals.
* MUST map reflection metadata into typed `Data` structures defined in the `System::Reflection` package.
* MUST treat the evaluated result of any `meta<TargetType>` query as a compile-time constant.

## Reflection Queries

* MUST support the `.Fields` reflection query on `Data` types.
* MUST evaluate `.Fields` to an array of field metadata structures (`FieldInfo`).
* MUST support the `.Variants` reflection query on types or fields declaring variants.
* MUST evaluate `.Variants` to an array of variant metadata structures (`VariantInfo`).
* MUST support the `.BaseType` reflection query on type guards.
* MUST evaluate `.BaseType` to a constant `String` containing the base type identifier.
* MUST support the `.Kind` reflection query across all valid types, evaluating to the type's structural category.
* MUST reject reflection queries not supported by the evaluated type kind at compile time.

```C#
User 
{
    $Id: Int;
    Name: String;
}

fields: [FieldInfo] = meta<User>.Fields;

for f in fields 
{
    Console.WriteLine(f.Name);
}

^Positive = Int where @ > 0;
base: String = meta<^Positive>.BaseType;
```
