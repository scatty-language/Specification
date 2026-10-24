# Generics

* MUST compile generic declarations using monomorphization.
* MUST guarantee at compile time that type arguments satisfy all specified interface constraints.

```C#
Cell<T> 
{
    Value: T;
}

PrinterData<T: ~Printable> 
{
    Item: T;

    PrintItem(@) => @.Item.Print();
}
```

## Static Dispatch

* MUST resolve all method calls on constrained generic parameters at compile time as direct, static dispatch calls.

```C#
ProcessItem<T: ~Entity, ~Printable>(item: T) 
{
    item.Update();
    item.Print();
}
```

## Generic Methods

* MAY declare independent generic parameters on individual methods.
* MUST NOT permit generic parameters of a method to shadow the generic parameters of the enclosing Data declaration.

```C#
Processor 
{
    Process<T: ~Printable>(item: T) 
    {
        item.Print();
    }
}
```
