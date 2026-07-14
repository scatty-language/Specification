# Memory Management

Coggy uses strict affine type semantics with ownership, move-by-default assignments, and deterministic destruction based on Non-Lexical Lifetimes (NLL).

## Ownership and Move Semantics

By default, every instance of Data has a single unique owner.

* **Move by Default**: Assigning a variable to another, or passing it as an argument, permanently moves ownership.
* **Invalidation**: Once moved, the original identifier is strictly INVALIDATED. Accessing or reassigning a moved identifier is FORBIDDEN.
* **Implicit Copy (`~ICopy`)**: If a Data type explicitly inherits the `~ICopy` abstract data, the compiler implicitly MUST performs a bitwise copy instead of a move. Standard numeric and logical types MUST implement `~ICopy`.
* **`~ICopy` Pointer Restrictions**: A Data type containing Sharp Pointers (`#`) as fields MUST NOT implement `~ICopy`. Types containing View Pointers (`&`) or Raw Pointers (`*`) MAY implement `~ICopy`.
* **Explicit Copy (`~IClone`)**: Heavy data structures MUST NOT implement `~ICopy`. They MAY implement `~IClone` to allow explicit duplication via a `.Clone()` method.

```C#
Consume(value: int) { }
ConsumeData(data: HeavyData) { }

x := 10;
Consume(x);
y := x; // Valid: 'int' implements ~ICopy, 'x' was copied, not moved

mut heavy := HeavyData.New();
otherHeavy := heavy; // 'heavy' is moved to 'otherHeavy'

ConsumeData(otherHeavy); // 'otherHeavy' is moved to method parameter

heavy = HeavyData.New(); 
// Error: 'heavy' is permanently invalidated after the first move

ValidCopy : ~ICopy    // Valid: View and raw pointers do not own memory
{
    View: int&;
    Raw: int*;
} 

InvalidCopy : ~ICopy  // Error: Data with sharp pointers MUST NOT implement ~ICopy
{
    Owned: int#;
}
```

## Pointers

Pointers define memory location and ownership contracts. Modifiers are applied on the right side of the type.

```EBNF
PointerType   ::= Type PointerMod ["?"]
PointerMod    ::= "#" | "&" | "*"
MethodPointer ::= "methodPointer" "<" ReturnType ("," Type)* ">"
```

### Sharp Pointer

A unique, non-null pointer representing absolute ownership of heap-allocated data.

* **Transfer of Ownership**: Passing a `#` moves both the instance and the responsibility to free the heap memory.
* **Heap Allocation Sugar**: Appending `#` as a postfix operator to any expression evaluates the expression, allocates strictly enough memory on the heap, moves the evaluated result into that memory, and returns a `T#` pointer.

```C#
// Allocates an 'int' on the heap, initializes it with '10', returns 'int#'
sharpIntPtr: int# = 10#; 

// Evaluates 'SomeData.New()', allocates memory on the heap, moves the instance, returns 'SomeData#'
sharpSomeDataPtr: SomeData# = SomeData.New()#;

// Also works with local variables
stackData := SomeData.New();
movedToHeapPtr := stackData#; // 'stackData' is moved to the heap
```

### View Pointer

A non-owning, non-null aliased pointer.

* **Non-owning**: Passing a `&` DOES NOT transfer ownership or destruction responsibility.
* **Exclusive Mutability**: To guarantee memory safety (e.g., preventing iterator invalidation during collection reallocation) and preserve invariant type refinements, strict aliasing limits apply:
  1. **Shared Read**: Multiple immutable view pointers (`&`) to the same instance MAY be active concurrently.
  2. **Exclusive Write**: Strictly ONLY ONE mutable view pointer (`mut&`) to an instance MAY be active at any given time.
  3. **Mutual Exclusion**: If a `mut&` is active, NO OTHER view pointers (mutable or immutable) to that instance can exist until the Last Use Place (LUP) of the `mut&`.
* **Owner Locking**: Creating any view pointer conceptually "locks" the original owner. The original owner instance MUST NOT be mutated, reassigned, or moved until the LUP of all active view pointers.
* **Deferred Destruction**: The compiler MUST guarantees the underlying memory remains alive until the LUP of the longest-living view pointer.

```C#
mut list := List.New();

view1: List& = &list;
view2: List& = &list;
Consume(view1, view2);

mut mutView: List& = &list;

badView: List& = &list;      // ERROR: Cannot create '&' while 'mut&' is active
mut badMut: List& = &list;   // ERROR: Cannot create a second 'mut&'
list.Add(10);                // ERROR: Owner 'list' is locked and cannot be mutated while borrowed

mutView.Add(5);              // Valid mutation via the exclusive view pointer
// LUP for mutView

// Owner now unlocked
list.Add(10);                // Valid: All view pointers are dead, owner is unlocked
```

### Pointer Modifiers

Pointers are strictly non-null by default. The `?` modifier creates an Option Pointer (e.g., `int#?`, `int&?`).

* Option Pointers MAY hold a `None` value.
* The compiler MUST use null-pointer optimization, ensuring the size of `T#?` equals `T#`.

```C#
sharpPtr: int# = 10#;
optionPtr: int#? = None;

viewPtr: int& = &sharpPtr;
mutView: mut int& = &sharpPtr;

ConsumeView(viewPtr);
```

## Lifetimes and Destruction

Coggy employs Non-Lexical Lifetimes (NLL). The compiler statically determines the Last Use Place (LUP) of an instance or its view pointers in the execution graph.

* **Deterministic Destruction**: The destructor of an instance MUST be injected immediately after its LUP, not at the end of the lexical block.
* **Deferred Destruction**: If an instance has active view pointers (`&`), the destruction of the instance's memory is deferred until immediately after the LUP of the longest-living view pointer.

### Destructor Generation and Contracts

* **Auto-Generation**: For standard Data and `#` pointers, the compiler AUTOMATICALLY generates the destructor and memory free instructions.
* **`~IOnDestroy`**: If auto-generation is sufficient, but custom pre-destruction logic is required, the Data MAY inherit the `~IOnDestroy` abstract data.
* **`~IDestroy`**: If a Data type contains raw pointers (`*`), auto-generation is STRICTLY DISABLED.

## Unsafe Memory and Raw Pointers

Raw pointers (`*`) and manual lifecycle control override the compiler's safety guarantees.

* **Context Requirement**: Any operation involving raw pointers or manual memory pinning MUST occur within an `unsafe` block or an `unsafe` method.
* **Raw Pointers (`*`)**: Unsafe, nullable, non-owning pointers. Creating a raw pointer from a `#` pointer does NOT defer the `#` pointer's destruction.
* **Method Pointers**: Function pointers (`methodPointer`) are inherently unsafe and MUST be invoked in an `unsafe` scope.

### Pin and Unpin

* `unpin(identifier)`: Instructs the compiler to REMOVE the destruction responsibility for the given identifier from the current scope. It will leak if not manually freed or pinned elsewhere.
* `pin(identifier)`: Instructs the compiler to ATTACH destruction responsibility for a raw or unpinned pointer to the current scope.

```C#
UnsafeOperation() unsafe
{
    sharpPtr: int# = 10#;
    
    // Raw pointer alias. Does not extend sharpPtr's lifetime
    rawPtr: int* = Unsafe.AsRawPointer(sharpPtr);
    
    // Unpin completely removes sharpPtr's auto-destruction
    unpin(sharpPtr);
    
    // Memory will leak here unless manually freed or re-pinned
}
```
