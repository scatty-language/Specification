# Data

Data declarations define memory layouts, state, and behaviors. By default, all `Data` types are value types.

Coggy uses a single, unified `Data` concept.

```EBNF
DataDecl        ::= ["~"] Identifier ["<" TypeArgs ">"] [":" ParentList] "{" Member* "}"
ParentList      ::= Identifier ("," Identifier)*

Member          ::= DataDecl | FieldDecl | VariantFieldDecl | MethodDecl | RefinementDecl
FieldDecl       ::= Identifier ":" Type ["=" Expression] ";"

Type            ::= Identifier ["<" TypeArgs ">"] TypeModifier*
TypeModifier    ::= "[]" | "*" | "?"
```

* **The `~` Modifier**: The only permitted left-side modifier for a top-level `Data` declaration is `~`, which strictly marks the type as Abstract Data.
* **Type Modifiers**: At the call site, types MAY be modified strictly by specified modifiers.

## Standard Data

A standard `Data` declaration defines a concrete, instantiable value type.

* **Allocation**: Instances are always allocated inline.
* **Inheritance**: Supports inheritance EXCLUSIVELY from Abstract Data (`~Data`). A standard Data type MUST NOT inherit from other standard Data types.
* **Initialization Order**: Default values MUST evaluate strictly before the initializer fully executes.
* **Memory Layout**:
  * Fields MUST be laid out sequentially in memory. The compiler MAY apply padding to satisfy alignment.
  * **Empty Data Size**: Any instantiable Data type declared without fields MUST occupy a minimum of 1 byte in memory.

## Nested Data Access

Data types can be nested within other Data to encapsulate helper structures or sub-components.

* **Scope Resolution**: To access a nested type, the `::` scope resolution operator MUST be used (`Outer::Inner`).
* **Instantiation & Calls**: Once resolved via `::`, standard initializer syntax (`Outer::Inner {}`) and static method calls (`Outer::Inner.Method()`) apply.
* **No Implicit Link**: Nested Data types are strictly lexically scoped. An instance of a nested type DOES NOT contain an implicit pointer to an instance of its outer type.

```C#
// Standard Data
Vector 
{
    X: int;
    Y: int;
}

Graph 
{
    Root: Node;

    // Nested declaration
    Node 
    {
        Id: int;

        // Recursive heap-allocation using 'Box'
        Left: Box<Graph::Node>;
        Right: Box<Graph::Node>;
    }
}

vector := Vector { X: 1, Y: 1 };

// Heap allocation via 'Box'
leaf := Graph::Node { Id: 1, Left: Box.Empty(), Right: Box.Empty() };
root := Graph::Node { Id: 0, Left: Box.New(leaf), Right: Box.Empty() };
```
