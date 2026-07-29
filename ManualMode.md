# Manual Mode

By default, Coggy manages dynamic heap memory via Automated Reference Counting (ARC). For domains requiring absolute, deterministic control over allocations and zero hidden overhead, Coggy provides Manual Mode.

Manual Mode operates at the Package Level. It transfers lifetime management of heap-allocated objects to the developer.

* **Safety Preservation**: Manual Mode is NOT `unsafe` mode. It does not allow raw pointers, unchecked memory arithmetic, or disabling bounds checking. It solely disables ARC and Copy-on-Write (CoW).
* **Package Boundary**: The mode is enabled by declaring `memory = "manual"` in the package's `package.toml` manifest.

## Implicit Type Substitution

When a package is compiled in Manual Mode, the compiler intrinsically alters the memory layout of all standard heap-backed types (`String`, `Type[]`, `[]Type`, `Box<T>`).

* **No ARC Tokens**: The compiler physically strips the reference counting metadata from the heap layout of these types.
* **Standard Library Mapping**: Identifiers for standard dynamic types are implicitly remapped to their non-ARC counterparts in the `System::Manual` package (e.g., `String` resolves to `System::Manual::String`).
* **Value Semantics (Deep Copy)**: Assigning a manual heap-backed type to a new variable (`a := b`) or passing it by value to a function performs an immediate Deep Copy to preserve logical independence. Developers MUST pass by `ref` to avoid allocations.

## `destroy` Operator

Manual Mode introduces the `destroy` statement to explicitly deallocate heap memory and release resources.

```EBNF
DestroyStmt ::= "destroy" Expression ";"
```

* **Target Constraint**: The `Expression` MUST evaluate to a manual heap-allocated type (e.g., `System::Manual::String`) OR a type implementing the `~` interface.
* **Execution**:
  1. If the type implements `~Disposable`, the `Dispose(ref @)` method is invoked.
  2. If the type is heap-allocated, the underlying memory is instantly deallocated.
* **ARC Prohibition**: Applying `destroy` to an ARC-managed type (within a standard package) is a STRICT compilation error. ARC memory MUST be managed implicitly.
* **Use-After-Free**: After a variable is passed to `destroy`, the compiler flags the identifier as poisoned. Any subsequent read or write to that variable in the current lexical scope is a compilation error.

```C#
// Inside a Manual Mode package
ProcessLog() 
{
    // Implicitly System::Manual::String. Heap allocated without ARC
    text := "Engine starting..."; 
    
    // Deep copy
    textCopy := text; 

    // Explicit deallocation
    destroy text;
    destroy textCopy;
    
    // ERROR: 'text' is poisoned
    Console.WriteLine(text); 
}
```

## Interoperability and Transformations

Because standard ARC types and Manual types have fundamentally different physical memory layouts, they are incompatible by default.

To pass data between an ARC package and a Manual package, the memory MUST be explicitly transformed using the `to` operator.

```C#
// Inside an ARC Mode package
use AudioEngine; // A package compiled in Manual Mode

App 
{
    Run() 
    {
        // Standard ARC-managed 'String'
        arcText: String = "Play Sound"; 
        
        // ERROR: Type mismatch. Cannot pass ARC String to a Manual 'String' parameter
        AudioEngine.Process(arcText); 

        // Physical memory transformation: 
        // Reads the ARC 'String', allocates a new manual 'String' buffer without ARC
        manualText := arcText to System::Manual::String;
        
        // Valid call
        AudioEngine.Process(manualText);
        
        // The calling ARC package owns the newly allocated manual 'String' 
        // and MUST clean it up
        destroy manualText; 
    }
}
```
