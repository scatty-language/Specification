# Collections

Coggy provides intrinsic support for statically sized arrays, fixed-size heap arrays, dynamically sized lists, and Structure of Arrays (SoA) memory layouts.

Heap-allocated collections (Arrays and Lists) are managed by Automated Reference Counting (ARC) with Copy-on-Write (CoW) semantics. Static arrays (`Type[N]`) are purely stack-allocated value types.

## Heap Arrays

The postfix `[]` modifier declares a standard, heap-allocated Array of Structures.

* **Fixed Capacity**: The size of a heap array is defined strictly at runtime during instantiation and CANNOT grow or shrink.
* **By-Reference**: Individual elements CAN be passed by reference.

```C#
Vector { X: Int; Y: Int; }

// Memory: [X, Y] [X, Y] [X, Y]
// Instantiated with a fixed size of 3
aos: Vector[] = Vector[].New(3);
aos[0] = Vector { X: 1, Y: 1 };

Process(ref aos[0]);
```

## Dynamic Lists

The postfix `[~]` modifier declares a heap-allocated, dynamically resizable List.

* **Dynamic Capacity**: The list automatically manages a hidden capacity and reallocates its internal buffer when it grows.
* **By-Reference**: Individual elements CAN be passed by reference.

```C#
list: Vector[~] = Vector[~].New();

// Dynamically grows the underlying buffer
list.Add(Vector { X: 1, Y: 1 });
list.Add(Vector { X: 2, Y: 2 });

Process(ref list[1]);
```

## Hashed Collections

Coggy provides intrinsic syntax for heap-allocated, hashed collections. Like dynamic lists, they are managed by Automated Reference Counting (ARC) with enforced Copy-on-Write (CoW) semantics.

### Hash Sets

The `[$]` postfix modifier declares a Hash Set: an unordered collection of unique elements based on their Identity hash.

* **Identity Requirement**: The element type MUST inherently support hashing. For custom `Data` types, the hash is intrinsically derived from fields marked with the `$` modifier, or all fields if no `$` modifier is present.
* **Uniqueness**: Adding an element that already exists (matching Identity) overwrites the existing entry or acts as a no-op.

```C#
// A hashed set of 'String'
activeUsers: String[$] = String[$].New();

activeUsers.Add("Alice");
activeUsers.Add("Bob");

if activeUsers.Contains("Alice") 
{
    Console.WriteLine("Found!");
}
```

### Hash Maps

The `[$KeyType]` postfix modifier declares a Hash Map. It logically represents a dynamically sized collection of `ValueType` elements, indexed by the Identity hash of a `KeyType`.

* **Key Identity**: The `KeyType` MUST support structural hashing.
* **Indexing**: Elements are accessed or mutated using standard bracket notation `[key]`.
* **Missing Keys**: Accessing a key that does not exist via direct bracket notation (`map[key]`) intrinsically throws a `KeyNotFoundException`. Developers MAY use standard methods (like `.TryGetValue()` or `.Contains()`) for safe access.

```C#
// A map of 'Int' keyed by 'String'
scores: Int[$String] = Int[$String].New();

// Insertion/Mutation
scores["Alice"] = 100;
scores["Bob"] = 80;

// Access
aliceScore := scores["Alice"];

// Safe access without throwing
if score := scores.TryGetValue("Charlie") 
{
    Console.WriteLine(score);
}
```

## Static Arrays

The postfix `[N]` modifier (where `N` is a positive, compile-time integer constant) declares a fixed-size static array.

* **Stack Allocation**: Static arrays are allocated inline, either on the call stack or physically embedded within an enclosing `Data` layout. They DO NOT use the heap and DO NOT use ARC.
* **Pure Value Semantics**: Assigning a static array to a new variable or passing it by value performs an immediate, full structural memory copy. CoW DOES NOT apply.
* **Initialization**: The array MUST be instantiated using the `{}` block syntax, providing expressions for its elements.

```C#
// Stack-allocated static array of size 2
x := Int[2] { 0, 0 };

// Full memory copy
y := x;
y[0] = 5;

Console.WriteLine(x[0]); // Output: 0

Matrix 
{
    // Physically embedded directly inside the Matrix memory layout
    Cells: Float[16]; 
}
```

## Structure of Arrays

The prefix modifiers `[]`, `[~]`, and `[N]` declare a Structure of Arrays. This instructs the compiler to intrinsically invert the structural memory layout of the specified type.

* **Deep Flattening**: The compiler recursively breaks down the base type (and any nested inline fields, structural fields, or array metadata) into independent, parallel contiguous arrays for each primitive data field.
* **Static SoA (`[N]Type`)**: Prefixing with a constant size allocates the parallel arrays inline on the stack, bypassing ARC and adhering to pure Value Semantics.
* **Target Restriction**: SoA prefixes CAN be applied to custom `Data` types and collections (e.g., `Int[]`), as they structurally possess internal fields. Applying SoA prefixes to scalar primitive types (e.g., `Int`, `Float`) is FORBIDDEN.

```C#
Entity 
{
    Id: Int;
    Position: Vector;
}

// Heap-Allocated Fixed SoA Array (Size 100)
fixedSoa: []Entity = []Entity.New(100);

// Heap-Allocated Dynamic SoA List
dynamicSoa: [~]Entity = [~]Entity.New();

// Stack-Allocated Static SoA (Size 16)
staticSoa := [16]Entity { ... };

// Valid: Flattens the internal struct of the Heap Array (token, length, ptr)
complexSoa: [~]Int[] = [~]Int[].New();
```

### Gather, Scatter, and Field Access

The compiler transparently translates standard indexing into the corresponding parallel array operations.

* **Field Access**: Accessing a specific field of an indexed element (`soa[i].Field`) directly reads or writes to the underlying independent array.
* **Scatter (Write)**: Assigning a whole object to an index (`soa[i] = instance`) automatically scatters the fields of the instance into the respective parallel arrays.
* **Gather (Read)**: Reading a whole object from an index (`val := soa[i]`) automatically gathers the fields from the parallel arrays and constructs a transient, stack-allocated instance of the `Data` type.

```C#
soa: [~]Entity = [~]Entity.New();
soa.Add(Entity { Id: 1, Position: Vector { X: 10, Y: 10 } });

// Field access
Console.WriteLine(soa[0].Id);

// Scatter: writes to independent arrays internally
soa[0] = Entity { Id: 2, Position: Vector { X: 20, Y: 20 } };

// Gather: creates a stack-allocated 'Entity' instance
snapshot := soa[0];
```

### The Reference Restriction

* **Whole Element Prohibition**: Passing an indexed SoA element via the `ref` keyword (`ref soa[i]`) is FORBIDDEN.
* **Field Exemption**: Passing an individual, primitive field of an indexed SoA element via the `ref` keyword (`ref soa[i].Id`) is ALLOWED.
* **By-Value Allowance**: Passing an indexed SoA element by value (`Process(soa[i])`) is ALLOWED.

```C#
MutateEntity(ref entity: Entity) { }
MutateId(ref id: Int) { }
ProcessEntity(entity: Entity) { }

soa: [~]Entity = [~]Entity.New();
soa.Add(Entity { Id: 1, Position: Vector { X: 10, Y: 10 } });

// ERROR: Cannot pass a scattered SoA element by reference
MutateEntity(ref soa[0]);

// Passing a single field by reference
MutateId(ref soa[0].Id);

// Passes by value (implicit gather on the stack)
ProcessEntity(soa[0]);
```

### CoW and SoA Iteration

SoA types adhere to Value Semantics. If a `[]Data` or `[~]Data` is reassigned (`soa2 := soa`), the parallel arrays share the same ARC buffer until a mutation occurs.

When iterating over an SoA collection, standard iterator loops natively perform the Gather operation.

```C#
// Iteration dynamically gathers elements by value
for entity in soa 
{
    Console.WriteLine(entity.Id);
}
```
