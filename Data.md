# Data

Data declarations define memory layouts and behaviors. Coggy uses a unified `Data` concept, distinguished by strictly positioned modifiers.

## Type Modifiers Position

* **Left-Side Modifiers**: Used at declaration. Define the fundamental kind and memory constraints of the Data.
* **Right-Side Modifiers**: Used at usage/call sites. Define how the data is stored or accessed in memory.

## Data Declarations & Left-Side Modifiers

```EBNF
DataIdentifier  ::= DataModifier* Identifier
DataModifier    ::= "%" | "~" | "&" | "#"
DataDecl        ::= DataIdentifier [":" ParentList] ["::" StateList] "{" Member* "}"
ParentList      ::= Identifier ("," Identifier)*

Member          ::= DataDecl | FieldDecl | FunctionDecl | RefinementDecl
FieldDecl       ::= Identifier ":" Type ["=" Expression] ";"

Type ::= DataIdentifier ["::" StateIdentifier] TypeModifier*
TypeModifier ::= "[]" | "#" | "&" | "*" | "?"
```

### Standard Data

Can be instantiated on the stack or heap. Supports single inheritance from other Standard Data or Abstract Data.

* **Memory Layout**: Fields MUST be laid out sequentially in memory. The compiler MAY apply padding to satisfy architecture alignment.
* **Initialization Order**: Default values MUST evaluate strictly before the initializer executes.

### Abstract Data

Defines an abstract base type.

* **No Initialization**: MUST NOT be directly initialization via `T {}`.
* **Object Slicing Protection**: Standard instances CANNOT be passed by value to an abstract type parameter and passed by references.

### Stack-Only Data

Strictly transient, stack-only data views.

* **Stack Enforcement**: Stack-Only Data MUST NOT escape the local stack frame. They CANNOT be allocated on the heap.
* **Composition Rules**: A Stack-Only Data type (`&T`) MAY only be declared as a field within another Stack-Only Data. It MUST NOT be a field in Standard Data or Dynamic Data.
* **Generics Restriction**: Stack-Only Data MUST NOT be passed as generic type arguments unless the generic parameter explicitly requires a stack-only constraint.
* **Lifecycle Restrictions**: Stack-Only Data MUST NOT implement a `~IDestroy` abstract data.
* **Inheritance**: MAY inherit from multiple Stack-Only Data, provided there are no overlapping field layouts. CANNOT inherit from Standard or Abstract Data.

### Heap-Only Data

Data explicitly constrained to the heap.

* **Allocation**: `#Type {}` implicitly allocates on the heap.
* **Restriction**: Using the `#` usage modifier on a heap-only type is an error.

### Enumeration Data

Enumerations are Tagged Unions fully integrated into the `Data` system. They define a closed set of variants while supporting all standard Data capabilities, including shared fields, properties, methods, and memory constraints.

```EBNF
VariantDecl     ::= Identifier ["(" PayloadList ")"] ["=" Expression] ","
PayloadList     ::= PayloadField ("," PayloadField)*
PayloadField    ::= Identifier ":" Type ["=" Expression]
```

* **Memory Layout**: The memory footprint of an enum instance is strictly the sum of: `Size(Shared Fields) + Size(Tag) + Max(Size(Variant Payloads))`.
* **Shared Members**: Enums MAY declare standard fields, properties, and methods. Shared fields are stored independently of the variant and are accessible unconditionally.
* **Inheritance Binding**: If an enum inherits from a base Data type (e.g., `int`), variants MAY implicitly provide an initialization expression (`= Expression`) conforming to that base type.
* **Virtual Behaviors**: If declared as abstract (`%~`), the enum MAY define `~Virtual` or `~Abstract` methods. Variants CANNOT override methods individually; overrides MUST be implemented by inheriting Data types.

```C#
// Heap-Only Enum inheriting from string
%#HttpError : string 
{
    // Shared field
    Timestamp: long;

    // Shared method
    PrintLog(@&) => Console.WriteLine($"{@.Timestamp}: {@}");

    // Variants
    NotFound = "404 Not Found",
    Timeout(RetryAfter: int) = "408 Request Timeout",
    Internal(Code: int) = "500 Internal Error",
}

// Standard Enum with shared fields
%Shape 
{
    Id: int;
    
    // Shared property
    IsValid: bool => @.Id > 0;

    Circle(Radius: float = 10.0),
    Rectangle(Width: int, Height: int),
    Point,
}
```

#### Variant Instantiation

Variants are instantiated using the scope resolution operator `::`.

The initializer MUST provide values for both the variant's payload (if any) and all shared fields of the enum.

```C#
circle := Shape::Circle { Id: 1, Radius: 5.0 };
timeoutError := HttpError::Timeout { Timestamp: 1620000000, RetryAfter: 30 }#;
```

#### Variant Refining and Casting

A specific enum variant acts as a strict subset of the enum type. Accessing a variant's payload is ONLY permitted after refining the instance to that specific variant type.

* **The `is` Operator**: Evaluates if the active tag matches the specified variant. Returns a `bool`.
* **The `as` Operator (Throwing Cast)**: Asserts the variant. Returns the refined variant type granting access to the payload. Throws a `#CastException` if the active tag does not match.
* **The `as!` Operator (Uncheked Cast)**: Returns the refined variant type granting access to the payload without check.
* **The `as?` Operator (Option Cast)**: Safely attempts the cast, returning `Option<Enum::Variant>`.

```C#
ProcessShape(shape: %Shape) 
{
    // Unconditional access to shared fields/methods
    Console.WriteLine(shape.Id);
    
    // shape.Radius; // ERROR: Payload 'Radius' is not accessible on unrefined '%Shape'

    // Boolean check
    if shape is Shape::Circle 
    {
        // Implicit refinement inside the block (if implemented by compiler)
        // or explicit manual cast:
        circle := shape as Shape::Circle;
        Console.WriteLine(circle.Radius); // Valid
    }

    // Pattern Matching
    match shape 
    {
        .Circle => Console.WriteLine(shape.Radius),
        .Rectangle => Console.WriteLine(shape.Width),
        _ => Console.WriteLine("No payload")
    }
}
```

## Inheritance and Virtual Functions

Any data type (except Stack-Only Data) MAY define inheritance and virtual behaviors using the `~` prefix on functions.

* **Virtual (`~Method()`)**: Has a body. Opens the method for overriding in descendants.
* **Abstract (`~Method();`)**: No body. MUST be overridden in descendants. The enclosing Data MUST be marked as `~`.
* **Sealed (`Method()`)**: No prefix. MUST NOT be overridden by descendants. If it overrides a parent method, it seals it.

## Nested Data Access

Data types can be nested within other Data to encapsulate helper structures or sub-components.

* **Scope Resolution**: To access a nested type, the `::` scope resolution operator MUST be used (`Outer::Inner`).
* **Instantiation & Calls**: Once resolved via `::`, initializer syntax (`Outer::Inner {}`) and static method calls (`Outer::Inner.Method()`) apply.

```C#
// Abstract Data
~Widget 
{
    Name: string;

    ~Draw()  // Virtual
    {
        Console.WriteLine("Drawing Widget");
    }
    
    ~Show(); // Abstract
}

// Standard Data inheriting Abstract Data
Button : ~Widget
{
    New(name: string) -> @
    {
        return Button {name};
    }

    Draw()  // Sealed Override
    {
        Console.WriteLine("Drawing Button");
    }

    ~Show() // Virtual Override
    {
        Console.WriteLine("Showing Button");
    }
}

// Invalid initialization
widget := ~Widget {"Base"}; // Error: ~Widget is abstract

ProcessValue(w: ~Widget) { }

button := Button.New("Submit");
heapButton := Button.New("Submit")#;

ProcessValue(button);       // Error: Object slicing
ProcessValue(heapButton);   // Valid: Pointer reference

// Heap-Only Data
#Exception 
{
    Message: string;
}

exception := #Exception {"Oops!"};         // Valid: Implicitly on heap
badException := new #Exception {"Oops!"}#; // Error: Sharp pointer to sharp pointer is invalid for '#T'
```
