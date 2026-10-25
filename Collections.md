# Collections

## Contextual and Indexed Initialization

* MAY omit the explicit `CollectionType` prefix before `{ ... }` when the expected collection type is statically known from context (explicitly typed variable declarations, assignments, method parameters, `return` statements, or structural field initializers).
* MUST reject an unqualified `{ ... }` collection initializer at compile time if the target type is inferred (`:=`) or ambiguous.
* MAY initialize array or list elements positionally (`{ expr1, expr2 }`) or by explicit index mapping (`{ index = expr }`).
* MAY provide a wildcard fallback entry (`_ = expr`) to initialize all remaining unmapped indices of a fixed-size or pre-allocated array.

```C#
// Contextual initialization without repeating '[Int]'
numbers: [Int] = { 1, 2, 3, 4 };

// Sparse index initialization with wildcard default fallback
lookup: [String ::: 32] = { 0 = "One", 20 = "Big", _ = "None" };
```

## Standard Arrays and Lists

* MUST allocate standard arrays (`[Type]`) contiguously with a fixed size determined at runtime instantiation.
* MUST manage a hidden dynamic capacity for lists (`[+ Type]`) to support resizing and mutation operations.

```C#
// Heap-allocated Array of size 3
aos: [Int] = [Int].New(3);
aos[0] = 10;

// Heap-allocated Dynamic List
list: [+ Int] = [+ Int] { 1, 2, 3 };
list.Add(4);
```

## Hashed Collections

* MUST enforce element uniqueness in Hash Sets (`[$ Type]`) using the element's Identity hash (`$`).
* MUST index Hash Maps (`[KeyType $ ValueType]`) using the key's Identity hash.
* MUST throw an exception upon accessing an unmapped key via the index operator (`map[key]`).
* MUST throw an exception (or trigger Abort/Panic in a non-throwing method) upon accessing an unmapped key via the index operator (`map[key]`).
* MAY use `.At(key)` to safely query a key and return `ValueType?` (`Option::Some` or `Option::None`) without throwing.

```C#
// A hashed set of Strings
activeUsers: [$ String] = [$ String].New();
activeUsers.Add("Alice");

// A map of Ints keyed by String
scores: [String $ Int] = [String $ Int].New();
scores["Alice"] = 100;
```

## Structure of Arrays

* MUST intrinsically invert the structural memory layout into parallel contiguous arrays for each field when the SoA modifier (`%`) is applied.
* MUST apply SoA modifiers ONLY to custom `Data` types or collections possessing internal fields.
* MUST NOT apply SoA modifiers to scalar primitives or interfaces.
* MAY combine the SoA modifier (`%`) with the dynamic list modifier (`+`) to declare dynamic SoA lists (`[+% Type]`).
* MUST recursively flatten the base type, nested embedded fields, structural fields, and array metadata into independent, parallel arrays.

```C#
Entity { Id: Int; PosX: Float; }

// Fixed-size SoA Array
soaArray: [% Entity] = [% Entity].New(100);

// Dynamic SoA List
soaList: [+% Entity] = [+% Entity].New();
soaList.Add(Entity { Id: 1, PosX: 0.0f });

// Zero-cost independent array access
id := soaList[0].Id;
```

### Gather, Scatter, Field Access

* MUST directly read from or write to the corresponding parallel array when evaluating a field access on an indexed SoA element (`soa[i].Field`).
* MUST implicitly scatter the fields of the assigned instance into the respective parallel arrays when writing to a whole indexed SoA element (`soa[i] = instance`).
* MUST implicitly gather the fields from the parallel arrays and construct a transient, stack-allocated instance of the `Data` type when reading a whole indexed SoA element (`val := soa[i]`).

```C#
soa := [+% Entity].New();
soa.Add(Entity { Id: 1, Position: Vector { X: 10, Y: 10 } });

// Field access
Console.WriteLine(soa[0].Id);

// Scatter: writes to independent arrays internally
soa[0] = Entity { Id: 2, Position: Vector { X: 20, Y: 20 } };

// Gather: creates a stack-allocated 'Entity' instance
snapshot := soa[0];
```

### The Reference Restriction

* MUST NOT pass a gathered SoA element by reference (`ref soa[i]`).
* MAY pass an individual primitive field of an indexed SoA element by reference (`ref soa[i].Id`).
* MAY pass an indexed SoA element by value.

```C#
MutateEntity(ref entity: Entity) { }
MutateId(ref id: Int) { }
ProcessEntity(entity: Entity) { }

soa: [+% Entity] = [+% Entity].New();
soa.Add(Entity { Id: 1, Position: Vector { X: 10, Y: 10 } });

// ERROR: Cannot pass a scattered SoA element by reference
MutateEntity(ref soa[0]);

// Passing a single field by reference
MutateId(ref soa[0].Id);

// Passes by value (implicit gather on the stack)
ProcessEntity(soa[0]);
```

### SoA Iteration

* MUST treat SoA collections as ARC-managed Owning Types with Copy-on-Write semantics; reassignment increments the strong reference count, and mutation detaches a deep copy of the parallel arrays if `strongCount > 1`.
* MUST implicitly perform a Gather operation during standard iterator loops.
* MUST copy the gathered instance by value (bitwise copy for Trivial Types, ARC retain for Owning Types).
* MUST implicitly gather each element into a temporary stack instance and invoke `Dispose(ref temp)` prior to releasing nested owning fields whenever a SoA collection (`[% Type]` or `[+% Type]`) containing a `~Disposable` element type is destroyed, cleared, or has an element removed or overwritten.

```C#
// Iteration dynamically gathers elements by value
for entity in soa 
{
    Console.WriteLine(entity.Id);
}
```

## Static and Fixed-Capacity Layouts (`:::`)

* MAY explicitly constrain the physical memory layout of a collection using the Physical Layout operator (`:::`) and a compile-time constant.
* MUST allocate static arrays (`[Type ::: N]`) directly inline on the stack or physically embedded within an enclosing `Data` layout.
* MUST pre-allocate the exact required memory for fixed-capacity dynamic collections (`[+ Type ::: N]`, `[$ Type ::: N]`) and guarantee zero runtime reallocations.
* MUST throw a `CapacityException` if an insertion operation exceeds the explicitly defined fixed capacity `N`.

```C#
// Stack-allocated static array
x := [Int ::: 2] { 0, 0 };

// Full O(N) memory copy, no CoW!
y := x; 
y[0] = 5;

// Fixed-capacity dynamic list. Will NEVER reallocate.
fixedList := [+ Int ::: 16].New();
fixedList.Add(1);

Matrix 
{
    // Physically embedded (16 contiguous floats)
    Cells: [Float ::: 16]; 
}

// Stack-allocated Static SoA!
// Memory: [Id*16] [PosX*16] directly on the stack
staticSoa := [% Entity ::: 16] { ... };
```

## Associated Arrays (`->`, `->!`, `->?`)

* MAY declare a fixed-size associated array keyed by the variants of a `Data` type using `[KeyType -> ValueType]`, `[KeyType ->! ValueType]`, or `[KeyType ->? ValueType]`.
* MUST require `KeyType` to be a tagged `Data` type (not `::: none`) that declares variants without payload fields and declares zero shared instance fields.
* MUST allocate associated arrays inline on the stack or physically embedded within an enclosing `Data` layout with a fixed capacity equal to the exact count of variants declared in `KeyType`.
* MUST map each variant of `KeyType` to its 0-based declaration ordinal (`0` through `VariantCount - 1`), regardless of whether `KeyType` declares explicit sparse or negative tag values (`:::`).
* MUST resolve constant variant keys (`.Variant`) to their corresponding 0-based ordinal at compile time with zero runtime lookup overhead.

```C#
Color 
{
    ::Red   ::: 100;
    ::Green ::: 200;
    ::Blue  ::: 300;
}

// Physical capacity is 3 (ordinals 0, 1, 2), NOT 301
weights: [Color -> Int] = { .Red = 10, .Green = 20, .Blue = 30 };

// Resolves '.Blue' to ordinal 2 at compile time
blueWeight := weights[.Blue];
```

### Exhaustive Associated Arrays (`[K -> V]`)

* MUST initialize every variant of `KeyType` when instantiating `[KeyType -> ValueType]`, or provide a wildcard fallback entry (`_ = Expression`) to populate all omitted variants.
* MUST NOT allocate a runtime validity bitmask for exhaustive associated arrays (`[KeyType -> ValueType]`).
* MUST evaluate an index access (`array[key]`) on `[KeyType -> ValueType]` infallibly to `ValueType`.

```C#
// Exhaustive via explicit variant coverage
rgbHex: [Color -> Int] = { .Red = 0xFF0000, .Green = 0x00FF00, .Blue = 0x0000FF };

// Exhaustive via wildcard fallback ('_'); '.Green' and '.Blue' are initialized to 0
redOnly: [Color -> Int] = { .Red = 100, _ = 0 };
```

### Non-Exhaustive Associated Arrays (`[K ->! V]` and `[K ->? V]`)

* MAY omit variants without a wildcard `_` fallback when initializing `[KeyType ->! ValueType]` or `[KeyType ->? ValueType]`.
* MUST encapsulate a fixed-size validity bitmask of `VariantCount` bits alongside the `VariantCount` element slots for `[KeyType ->! ValueType]` and `[KeyType ->? ValueType]`.
* MUST set the bitmask bit corresponding to a variant's ordinal to `1` when that variant is initialized or assigned (`array[key] = value`), and `0` when omitted.
* MUST verify the validity bit for `key`'s ordinal upon reading from `[KeyType ->! ValueType]` (`val := array[key]`), returning the stored `ValueType` if the bit is `1`, or immediately triggering runtime Abort or Fatal Panic mechanics if the bit is `0`.
* MUST evaluate an index read (`array[key]`) on `[KeyType ->? ValueType]` to `ValueType?` (`Option<ValueType>`), yielding `Option::Some` if the validity bit for `key`'s ordinal is `1`, or `Option::None` if the bit is `0`.

```C#
// Non-exhaustive panic-on-missing array: capacity is 3 slots + 3-bit mask (0b101)
strictPrices: [Color ->! Int] = { .Red = 100, .Blue = 300 };

r := strictPrices[.Red];   // Evaluates to 100 (bit 0 is 1)
g := strictPrices[.Green]; // Aborts/Panics immediately (bit 1 is 0)

// Dynamically populating an omitted variant sets bit 1 in the bitmask to 1
strictPrices[.Green] = 200;

// Non-exhaustive optional array
safePrices: [Color ->? Int] = { .Red = 100, .Blue = 300 };

if greenPrice := safePrices[.Green] 
{
    Console.WriteLine(greenPrice);
} 
else 
{
    Console.WriteLine("Green price is not configured");
}
```
