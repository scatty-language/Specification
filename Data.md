# Data

Data declarations define memory layouts, state, and behaviors. By default, all `Data` types are value types.

Coggy uses a single, unified `Data` concept.

```EBNF
DataDecl        ::= Identifier ["<" TypeArgs ">"] [":" ParentList] "{" Member* "}"
ParentList      ::= Identifier ("," Identifier)*

Member          ::= FieldDecl | ConstDecl | InlineFieldDecl | VariantFieldDecl | PropertyDecl | MethodDecl

FieldDecl       ::= [FieldModifier] Identifier ":" Type ["=" Expression] ";"
FieldModifier   ::= "$" | "#" | "#$"
```

## Standard Data

A standard `Data` declaration defines a concrete, instantiable value type.

* **Allocation**: Instances are always allocated inline.
* **Initialization Order**: Default values MUST evaluate before the initializer fully executes.
* **Structural Equality**: The compiler MUST generates default equality operators (`==` and `!=`) for all Standard Data types. These operators determine equality by performing a field-by-field structural comparison.
* **Memory Layout**:
  * Fields MUST be laid out sequentially in memory. The compiler MAY apply padding to satisfy alignment.
  * **Empty Data Size**: Any instantiable Data type declared without fields MUST occupy a minimum of 1 byte in memory.
* **Flat Topology**: All top level members MUST be declared ONLY at the top level of a file.
