# Data

Data declarations define memory layouts and behaviors. Coggy uses a unified Data concept, distinguished by strictly positioned modifiers.

## Type Modifiers Position

* **Left-Side Modifiers**: Used at declaration. Define the fundamental kind and memory constraints of the Data.
* **Right-Side Modifiers**: Used at usage/call sites. Define how the data is stored or accessed in memory.

## Data Declarations & Left-Side Modifiers

```EBNF
DataIdentifier  ::= DataModifier* Identifier
DataModifier    ::= "~" | "&" | "#"
DataDecl        ::= DataIdentifier [":" ParentList] "{" Member* "}"
ParentList      ::= Identifier ("," Identifier)*

Member          ::= DataDecl | FieldDecl | VariantFieldDecl | MethodDecl | RefinementDecl
FieldDecl       ::= Identifier ":" Type ["=" Expression] ";"

Type            ::= DataIdentifier TypeModifier*
TypeModifier    ::= "[]" | "#" | "&" | "*" | "?"
```

### Standard Data

Can be instantiated on the stack or heap.

* **Inheritance**: Supports inheritance exclusively from Abstract Data (`~Data`). It MUST NOT inherit from other Standard, Stack-Only, or Heap-Only Data.
* **Memory Layout**: Fields MUST be laid out sequentially in memory. The compiler MAY apply padding to satisfy architecture alignment.
* **Initialization Order**: Default values MUST evaluate strictly before the initializer executes.

### Stack-Only Data

Strictly transient, stack-only data views.

* **Stack Enforcement**: Stack-Only Data MUST NOT escape the local stack frame. They CANNOT be allocated on the heap.
* **Composition Rules**: A Stack-Only Data type (`&T`) MAY only be declared as a field within another Stack-Only Data. It MUST NOT be a field in Standard Data or Heap-Only Data.
* **Generics Restriction**: Stack-Only Data MUST NOT be passed as generic type arguments unless the generic parameter explicitly requires a stack-only constraint.
* **Lifecycle Restrictions**: Stack-Only Data MUST NOT implement a `~IDestroy` abstract data.
* **Inheritance**: MAY inherit from multiple Stack-Only Data, provided there are no overlapping field layouts. CANNOT inherit from Standard or Abstract Data.

### Heap-Only Data

Data explicitly constrained to the heap.

* **Allocation**: Instantiating `#Type {}` implicitly allocates memory on the heap and returns a sharp pointer (`#Type#`).
* **Restriction**: Using the `#` usage modifier on a heap-only type at the call site (e.g., `#Type{}#`) is a compilation error (no sharp pointer to a sharp pointer).

## Nested Data Access

Data types can be nested within other Data to encapsulate helper structures or sub-components.

* **Scope Resolution**: To access a nested type, the `::` scope resolution operator MUST be used (`Outer::Inner`).
* **Instantiation & Calls**: Once resolved via `::`, initializer syntax (`Outer::Inner {}`) and static method calls (`Outer::Inner.Method()`) apply.

```C#
// Standard Data
Vector 
{
    X: int;
    Y: int;
}

// Heap-Only Data
#Exception 
{
    Message: string;
}

// Stack-Only Data
&Span 
{
    _ptr: int*;
    Length: int;
}

vector := Vector { X: 1, Y: 1 };

exception := #Exception { Message: "Oops!" }; // Implicitly returns #Exception#
// bad := #Exception { Message: "Oops!" }#;   // Error: '#' modifier on '#Data'

// Valid stack-only allocation
span := &Span { _ptr: null, Length: 0 };
```
