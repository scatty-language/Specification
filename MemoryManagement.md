# Memory Management

Coggy employs Value Semantics with deterministic memory management.

## Value Semantics and Independence

All variables are fundamentally treated as independent values. Variables are mutable by default.

* **Logical Independence**: Assigning a variable to another (`a := b`) or passing a variable to a function by value conceptually creates an independent, isolated copy of the data.
* **Mutation Isolation**: Mutating the copy MUST NOT affect the original variable. Shared mutable state is FORBIDDEN.

```C#
vectorA := Vector.New(0, 0);
vectorA.X = 1;

vectorB := vectorA; // Conceptually an independent copy
vectorB = 2;        // Mutates vectorB

Console.WriteLine(vectorA.X); // Output: 1
Console.WriteLine(vectorB.X); // Output: 2
```

## Stack Allocation and Copy-on-Write

* **Stack Allocation**: All `Data` are structurally allocated directly on the stack by default.
* **Automated Reference Counting (ARC)**: Types that require dynamic sizing (e.g., `[~]`, `String`) internally manage a heap buffer using an ARC token. When such a type is copied (`listB := listA`), the compiler increments the ARC counter and does NOT duplicate the heap memory.
* **Copy-on-Write (CoW)**: When a mutation is invoked on a type backed by ARC, the type MUST check the reference count.
  * If `ARC == 1`, the memory is uniquely owned and mutated in-place.
  * If `ARC > 1`, the type allocates a new heap buffer, duplicates the data, decrements the original ARC, and mutates the new buffer.

## In-Place Mutation

To mutate an original variable inside a function without triggering CoW or creating logical copies, it MUST be passed by reference using the `ref` keyword.

* **Explicit Intent**: Both the function declaration and the call site MUST explicitly use the `ref` keyword.

```C#
// The function declares it will mutate the original 'Vector'
SetToZero(ref vector: Vector) 
{
    vector.X = 0;
    vector.Y = 0;
}

vector := Vector.New(1 ,1);
SetToZero(ref vector);
```

### Law of Exclusivity

* **Exclusivity Rule**: A specific memory location passed as a `ref` argument MUST NOT be aliased by any other parameter (either by `ref` or by value) in the exact same function call.
* **Disjoint Fields Exemption**: Distinct structural fields of the same `Data` instance represent non-overlapping memory locations. The compiler MUST allow them to be passed simultaneously as separate `ref` arguments.
* **Overlapping Memory Prohibition**: Passing a parent object (by value or `ref`) aliases all of its internal fields. Passing a parent instance and one of its nested fields simultaneously is FORBIDDEN.

```C#
ProcessValues(ref a: Int, ref b: Int) { }
ProcessMixed(ref obj: Vector, ref val: Int) { }

vector := Vector.New(0, 0);

// ERROR: 'vector' is passed by 'ref' and by value simultaneously
Process(ref vector, vector); 

// VALID: 'X' and 'Y' are disjoint memory locations
ProcessValues(ref vector.X, ref vector.Y);

// ERROR: 'vector' memory completely overlaps with 'vector.X'
ProcessMixed(ref vector, ref vector.X); 
```

## Heap Allocation and Shared State

Sharing mutable state requires explicit intent via the Standard Library.

* **`Box<T>`**: Used to explicitly force a stack-allocated `Data` type onto the heap. This is required for defining recursive data structures.
* **`Shared<T>`**: Used to represent Shared Mutable State. `Shared<T>` disables CoW and allows multiple owners to hold a reference to the exact same heap allocation and mutate it.
* **Concurrency**: In multi-threaded contexts, `Shared<T>` is disallowed unless it is specifically a thread-safe variant like `AtomicShared<T>` or `Mutex<T>`.

```C#
Node 
{
    Value: Int;
    Left: Box<Node>;
    Right: Box<Node>;
}

logger := Shared.New(Logger.New());

parser := Parser.New(logger);     // parser holds a reference
analyzer := Analyzer.New(logger); // analyzer holds the SAME reference
```

## Resource Cleanup and Disposal

The compiler inherently manages memory for all standard types. Explicit resource cleanup is reserved for types wrapping unmanaged or `unsafe` resources (e.g., raw pointers, OS handles).

Types requiring custom cleanup MUST implement the `~Disposable` standard interface.

```C#
~Disposable 
{
    ~Dispose(ref @);
}
```

When a variable implementing `~Disposable` exits its lexical scope (via normal block termination or exception unwinding), the compiler implicitly injects a call to `Dispose(ref @)`. The invocation depends on the type's underlying memory model:

* **Stack-Allocated Types (No ARC)**: `Dispose(ref @)` is unconditionally invoked at the end of the scope.
  * **Return Exemption**: If the variable is returned to the caller, the cleanup invocation MUST be omitted.
* **ARC-Backed Types**: The ARC counter is decremented upon scope exit. `Dispose(ref @)` is invoked ONLY if the reference count reaches exactly `0`.

```C#
FileHandle : ~Disposable 
{
    _rawDescriptor: Int; // OS-level unmanaged resource

    ~Dispose(ref @) 
    {
        OS.Close(@._rawDescriptor);
    }
}

ProcessFile() throws 
{
    file := FileHandle { _rawDescriptor: OS.Open("data.txt") };
    
    // If 'Parse' throws an exception, the stack unwinds,
    // 'file' goes out of scope, and 'Dispose' is implicitly called
    Parse(file); 
    
    // Normal exit: 'Dispose' is implicitly called
}

CreateFile() -> FileHandle 
{
    newFile := FileHandle { _rawDescriptor: OS.Open("new.txt") };
    
    // 'Dispose' is NOT called here
    return newFile; 
}
```
