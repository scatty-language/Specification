# Memory Management

Coggy employs Value Semantics with strictly deterministic memory management.

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
* **Automated Reference Counting (ARC)**: Types that require dynamic sizing (e.g., `List`, `string`) internally manage a heap buffer using an ARC token. When such a type is copied (`listB := listA`), the compiler increments the ARC counter and does NOT duplicate the heap memory.
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

* **Exclusivity Rule**: If a variable is passed as a `ref` argument to a function, it MUST NOT be passed to any other parameter (either by `ref` or by value) in the exact same function call.

```C#
Process(ref a: Vector, b: Vector) { }

vector := Vector.New(0, 0);

// ERROR: 'vector' is passed by 'ref' and by value simultaneously
Process(ref vector, vector); 

// ERROR: 'vector' is passed by 'ref' to overlapping parameters
Process(ref vector, ref vector);
```

## Heap Allocation and Shared State

Sharing mutable state requires explicit intent via the Standard Library.

* **`Box<T>`**: Used to explicitly force a stack-allocated `Data` type onto the heap. This is required for defining recursive data structures.
* **`Shared<T>`**: Used to represent Shared Mutable State. `Shared<T>` disables CoW and allows multiple owners to hold a reference to the exact same heap allocation and mutate it.
* **Concurrency**: In multi-threaded contexts, `Shared<T>` is strictly disallowed unless it is specifically a thread-safe variant like `AtomicShared<T>` or `Mutex<T>`.

```C#
Node 
{
    Value: int;
    Left: Box<Node>;
    Right: Box<Node>;
}

logger := Shared.New(Logger.New());

parser := Parser.New(logger);     // parser holds a reference
analyzer := Analyzer.New(logger); // analyzer holds the SAME reference
```
