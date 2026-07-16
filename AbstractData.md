# Abstract Data

Abstract Data defines behavioral contracts, polymorphic interfaces, and inheritable memory states.

Abstract Data is declared using the `~` left-side modifier.

* **No Initialization**: Abstract Data MUST NOT be directly instantiated via the initializer syntax (`~Type {}`).
* **Object Slicing Protection**: Standard Data instances CANNOT be passed by value to an abstract type parameter. To utilize runtime polymorphism, Abstract Data MUST strictly be passed by pointers (`~Data&`/`~Data#`/`~Data*`).

## Abstraction Model

Inheritance capabilities are dictated by the presence of data fields.

### Stateless Abstract Data

If an Abstract Data type contains NO structural fields (only methods, properties, or constants), it has a memory size of zero.

* **Multiple Inheritance**: A Data type MAY inherit from an UNLIMITED number of stateless Abstract Data.

### Stateful Abstract Data

If an Abstract Data type contains at least one field, it dictates a memory layout.

* **Single Inheritance**: A Data type MAY inherit from STRICTLY ONE stateful Abstract Data.

```C#
// Stateless: Acts as an interface (no memory layout)
~IPrintable 
{
    ~Print(@&); // Abstract method
}

// Stateful: Acts as an abstract base class
~Widget 
{
    Name: string;
    ~Draw(@&);
}

// Button inherits ONE stateful '~Widget', and MULTIPLE stateless contracts
Button : ~Widget, ~IPrintable
{
    ~Draw(@&) => Console.WriteLine("Draw Button");
    ~Print(@&) => Console.WriteLine(@.Name);
}
```

## Inheritance and Virtual Methods

Abstract Data and their descendants use the `~` prefix on methods to define polymorphic behaviors.

* **Abstract (`~Method();`)**: Declared without a body. MUST be overridden in descendants. The enclosing Data MUST be marked as abstract (`~`).
* **Virtual (`~Method() {}`)**: Declared with a body. Opens the method for overriding in descendants.
* **Sealed (`Method() {}`)**: Declared without the `~` prefix. MUST NOT be overridden by descendants. If it overrides a parent method, it strictly seals it, terminating the virtual chain.

```C#
~BaseBehavior
{
    // Abstract
    ~Execute(mut @); 
    
    // Virtual
    ~Log(@&) 
    {
        Console.WriteLine("Base");
    }
}

StandardBehavior : ~BaseBehavior
{
    // Sealed override: Cannot be overridden further
    Execute(mut @) 
    {
        
    }

    // Virtual override: Leaves the method open for future descendants
    ~Log(@&) 
    {
        Console.WriteLine("Standard");
    }
}
```

## Polymorphic Dispatch

When an instance is bound to an Abstract Data pointer (`~Data&`/`~Data#`/`~Data*`), method calls are resolved dynamically at runtime.

```C#
ProcessDynamic(widget: ~Widget&) 
{
    widget.Draw(); // Dynamic dispatch via vtable
}

ProcessValue(w: ~Widget) { } // ERROR: Cannot pass Abstract Data by value

button := Button { Name: "Submit" };

ProcessDynamic(&button);
```
