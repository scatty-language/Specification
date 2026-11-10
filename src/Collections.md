# Collections

## Contextual and Indexed Initialization

* MAY omit the explicit `CollectionType` prefix before `{ ... }` when the expected collection type is statically known from context (explicitly typed variable declarations, assignments, method parameters, `return` statements, or structural field initializers).
* MUST reject an unqualified `{ ... }` collection initializer at compile time if the target type is inferred (`:=`) or ambiguous.
* MAY initialize collection elements positionally (`{ expr1, expr2 }`) or by explicit key/index mapping (`{ key = expr }`).
* MAY provide a wildcard fallback entry (`_ = expr`) to initialize all remaining unmapped indices of a fixed-size array, inline array, or associated array.

```scatty
// Contextual initialization without repeating '[Int]'
numbers: [Int] = { 1, 2, 3, 4 };

// Sparse index initialization with wildcard default fallback
lookup: [String; 32] = { 0 = "One", 20 = "Big", _ = "None" };
```

## Dynamic Lists and Fixed-Size Arrays

* MUST manage a hidden dynamic capacity for lists (`[Type]`) to support resizing and mutation operations.
* MUST allocate fixed-size heap arrays (`Array<Type>`) contiguously with an exact element count determined at runtime instantiation (`Array<Type>.New(count)`) and without dynamic capacity overhead.
* MUST NOT resize or append elements to an `Array<Type>` instance after instantiation.

```scatty
// Heap-allocated Dynamic List
list: [Int] = [Int] { 1, 2, 3 };
list.Add(4);

// Heap-allocated Fixed-Size Array of size 3
arr := Array<Int>.New(3);
arr[0] = 10;
```

## Hashed Collections

* MUST enforce element uniqueness in Hash Sets (`[$: Type]`) using the element's Identity hash.
* MUST index Hash Maps (`[KeyType: ValueType]`) using the key's Identity hash.
* MUST throw an exception (or trigger Abort/Panic in a non-throwing method) upon accessing an unmapped key via the index operator (`map[key]`).
* MAY use `.At(key)` to safely query a key and return `ValueType?` (`Option::Some` or `Option::None`) without throwing.

```scatty
// A hashed set of Strings
activeUsers: [$: String] = [$: String].New();
activeUsers.Add("Alice");

// A map of Ints keyed by String
scores: [String: Int] = [String: Int].New();
scores["Alice"] = 100;
```

## Structure of Arrays

* MAY apply the postfix `soa` type modifier to any linear collection type (`[Type] soa`, `[Type; N] soa`, `Array<Type> soa`, or `InlineArray<Type> soa`).
* MUST intrinsically invert the structural memory layout into parallel contiguous arrays for each field when the `soa` modifier is applied.
* MUST apply the `soa` modifier ONLY to collections whose element type is a custom `Data` type possessing internal fields.
* MUST NOT apply the `soa` modifier to non-collection types, hashed collections (`[$: Type]`, `[KeyType: ValueType]`), associated arrays (`[KeyType -> ValueType]`), or collections of scalar primitives or interfaces.
* MUST recursively flatten the base element type, nested embedded fields, structural fields, and array metadata into independent, parallel arrays.

```scatty
Entity { Id: Int; PosX: Float; }

// Fixed-size SoA Array
soaArray := (Array<Entity> soa).New(100);

// Dynamic SoA List
soaList: [Entity] soa = ([Entity] soa).New();
soaList.Add(Entity { Id: 1, PosX: 0.0f });

// Zero-cost independent array access
id := soaList[0].Id;
```

### Gather, Scatter, Field Access

* MUST directly read from or write to the corresponding parallel array when evaluating a field access on an indexed `soa` element (`soaCol[i].Field`).
* MUST implicitly scatter the fields of the assigned instance into the respective parallel arrays when writing to a whole indexed `soa` element (`soaCol[i] = instance`).
* MUST implicitly gather the fields from the parallel arrays and construct a transient, stack-allocated instance of the `Data` type when reading a whole indexed `soa` element (`val := soaCol[i]`).

```scatty
soaCol := ([Entity] soa).New();
soaCol.Add(Entity { Id: 1, Position: Vector { X: 10, Y: 10 } });

// Field access
Console.WriteLine(soaCol[0].Id);

// Scatter: writes to independent arrays internally
soaCol[0] = Entity { Id: 2, Position: Vector { X: 20, Y: 20 } };

// Gather: creates a stack-allocated 'Entity' instance
snapshot := soaCol[0];
```

### The Reference Restriction

* MUST NOT pass a gathered `soa` element by reference (`ref soaCol[i]`).
* MAY pass an individual primitive field of an indexed `soa` element by reference (`ref soaCol[i].Id`).
* MAY pass an indexed `soa` element by value.

```scatty
MutateEntity(ref entity: Entity) { }
MutateId(ref id: Int) { }
ProcessEntity(entity: Entity) { }

soaCol: [Entity] soa = ([Entity] soa).New();
soaCol.Add(Entity { Id: 1, Position: Vector { X: 10, Y: 10 } });

// ERROR: Cannot pass a scattered SoA element by reference
MutateEntity(ref soaCol[0]);

// Passing a single field by reference
MutateId(ref soaCol[0].Id);

// Passes by value (implicit gather on the stack)
ProcessEntity(soaCol[0]);
```

### SoA Iteration

* MUST treat heap-allocated `soa` collections (`[Type] soa` and `Array<Type> soa`) as ARC-managed Owning Types with Copy-on-Write semantics; reassignment increments the strong reference count, and mutation detaches a deep copy of the parallel arrays if `strongCount > 1`.
* MUST implicitly perform a Gather operation during standard iterator loops.
* MUST copy the gathered instance by value (bitwise copy for Trivial Types, ARC retain for Owning Types).
* MUST implicitly gather each element into a temporary stack instance and invoke `Dispose(ref temp)` prior to releasing nested owning fields whenever a `soa` collection containing a `~Disposable` element type is destroyed, cleared, or has an element removed or overwritten.

```scatty
// Iteration dynamically gathers elements by value
entity in soaCol 
{
    Console.WriteLine(entity.Id);
}
```

## Inline Arrays and Fixed-Capacity Collections

* MAY specify a fixed maximum capacity `N` on bracketed collections using the `; N` syntax (`[Type; N]`, `[$: Type; N]`, `[KeyType: ValueType; N]`), where `N` is a compile-time integer constant.
* MUST pre-allocate the exact required memory for fixed-capacity collections (`[Type; N]`, `[$: Type; N]`, `[KeyType: ValueType; N]`) and guarantee zero runtime reallocations.
* MUST throw a `CapacityException` if an insertion operation exceeds the explicitly defined fixed capacity `N`.
* MAY instantiate an inline static array using `InlineArray<Type>.New(N)`, where `N` MUST evaluate to a compile-time integer constant.
* MUST allocate `InlineArray<Type>` directly inline on the stack or physically embedded within an enclosing `Data` layout.
* MUST provide a default initialization expression (`Field: InlineArray<Type> = InlineArray<Type>.New(N);`) when declaring an `InlineArray<Type>` as a structural field of a `Data` type.

```scatty
// Stack-allocated inline array of size 2
x := InlineArray<Int>.New(2);
x[0] = 0;
x[1] = 0;

// Full O(N) bitwise copy, no CoW
y := x; 
y[0] = 5;

// Fixed-capacity dynamic list. Will NEVER reallocate
fixedList := [Int; 16].New();
fixedList.Add(1);

// Fixed-capacity map
fixedMap := [String: Int; 8].New();
fixedMap["A"] = 1;

Matrix 
{
    // Physically embedded (16 contiguous floats)
    Cells: InlineArray<Float> = .New(16); 
}

// Fixed-capacity SoA list
staticSoa: [Entity; 16] soa = { ... };
```

## Associated Arrays

* MAY declare a fixed-size associated array keyed by the variants of a `Data` type using `[KeyType -> ValueType]`.
* MUST require `KeyType` to be a tagged `Data` type (not `tag none`) that declares variants without payload fields and declares zero shared instance fields.
* MUST allocate associated arrays inline on the stack or physically embedded within an enclosing `Data` layout with a fixed capacity equal to the exact count of variants declared in `KeyType`.
* MUST map each variant of `KeyType` to its 0-based declaration ordinal (`0` through `VariantCount - 1`), regardless of whether `KeyType` declares explicit sparse or negative tag values (`(=)`).
* MUST resolve constant variant keys (`.Variant`) to their corresponding 0-based ordinal at compile time with zero runtime lookup overhead.
* MUST evaluate an index access (`array[key]`) on `[KeyType -> ValueType]` infallibly to `ValueType`.
* MUST initialize every variant of `KeyType` when instantiating `[KeyType -> ValueType]` for a non-optional `ValueType`, or provide a wildcard fallback entry (`_ = Expression`) to populate all omitted variants.
* MAY omit variants without a wildcard `_` fallback when `ValueType` is an Option type (`[KeyType -> T?]`), implicitly initializing all omitted variant slots to `Option::None` (`none`).

```scatty
Color 
{
    ::Red   = 100;
    ::Green = 200;
    ::Blue  = 300;
}

// Physical capacity is 3 (ordinals 0, 1, 2), NOT 301
weights: [Color -> Int] = { .Red = 10, .Green = 20, .Blue = 30 };

// Resolves '.Blue' to ordinal 2 at compile time
blueWeight := weights[.Blue];

// Exhaustive via wildcard fallback ('_'); '.Green' and '.Blue' are initialized to 0
redOnly: [Color -> Int] = { .Red = 100, _ = 0 };

// Non-exhaustive via Option value type ('Int?'); omitted '.Green' defaults to 'Option::None'
prices: [Color -> Int?] = { .Red = 100, .Blue = 300 };

r := prices[.Red]!;   // Evaluates to 100
g := prices[.Green]!; // Aborts/Panics immediately because '.Green' is 'Option::None'

may greenPrice := prices[.Green] => Console.WriteLine(greenPrice);
or => Console.WriteLine("Green price is not configured");
```

## Slices

* MAY declare a read-only slice type using `[Type..]` and a mutable slice type using `[ref Type..]`.
* MUST represent a slice (`[Type..]` or `[ref Type..]`) as a contiguous, non-owning view over a sequence of `Type` elements without its own heap allocation or ARC Control Block.
* MAY construct a read-only slice (`[Type..]`) by applying a range index expression (`col[start..end]`, `col[start..]`, `col[..end]`, or `col[..]`) to an instance of `[Type]`, `[Type; N]`, `Array<Type>`, `InlineArray<Type>`, `[Type..]`, or `[ref Type..]`.
* MAY construct a read-only byte slice (`[Char..]`) by applying a range index expression to a `String`, and a read-only rune slice (`[Rune..]`) by applying a range index expression to a `RuneString`.
* MAY construct a mutable slice (`[ref Type..]`) by applying a `ref` range index expression (`col[ref start..end]`, `col[ref start..]`, `col[ref ..end]`, or `col[ref ..]`) to a mutable lvalue of `[Type]`, `[Type; N]`, `Array<Type>`, `InlineArray<Type>`, `[ref Type..]`, or `RuneString`.
* MUST NOT construct a mutable byte slice (`[ref Char..]`) from a `String` outside of an `unsafe` block or `unsafe` method.
* MUST NOT apply range slicing to `soa` collections, hashed collections (`[$: Type]`, `[KeyType: ValueType]`), or associated arrays (`[KeyType -> ValueType]`).
* MUST NOT implicitly convert `[Type]`, `[Type; N]`, `Array<Type>`, `InlineArray<Type>`, `String`, or `RuneString` to `[Type..]` or `[ref Type..]`; an explicit slice expression (`col[..]` or `col[ref ..]`) MUST be used.
* MAY implicitly downgrade a mutable slice (`[ref Type..]`) to a read-only slice (`[Type..]`).

```scatty
numbers := [Int] { 10, 20, 30, 40, 50 };

// Read-only slices ('[Int..]')
middle := numbers[1..4];   // Views [20, 30, 40]
prefix := numbers[..2];    // Views [10, 20]
suffix := numbers[3..];    // Views [40, 50]
all    := numbers[..];     // Views all elements

// Mutable slice ('[ref Int..]')
mutMiddle := numbers[ref 1..4];
mutMiddle[0] = 99;         // Mutates 'numbers[1]' in-place

// String slicing yields '[Char..]'
text := "Hello, Scatty!";
helloBytes: [Char..] = text[0..5];
```

### Slice Bounds and Operations

* MUST evaluate the lower bound `start` as inclusive (defaulting to `0` when omitted) and the upper bound `end` as exclusive (defaulting to the source collection's element count when omitted).
* MUST throw an exception (or immediately trigger Abort or Fatal Panic mechanics in a non-throwing context) if `start < 0`, `end < start`, or `end` exceeds the source collection's element count.
* MUST evaluate a range slice where `start == end` and `0 <= start <= Count` to a valid empty slice with a length of `0`.
* MUST NOT mutate elements, assign to an index (`slice[i] = val`), invoke `ref @` methods on elements, or borrow elements via `ref` (`ref slice[i]` or `ref item in slice`) on a read-only slice (`[Type..]`).
* MAY mutate elements in-place, assign to an index (`slice[i] = val`), invoke `ref @` methods on elements, and borrow elements via `ref` (`ref slice[i]` or `ref item in slice`) on a mutable slice (`[ref Type..]`).
* MUST NOT resize, append to, or remove elements from `[Type..]` or `[ref Type..]`.
* MUST evaluate the strong reference count of an Owning Type source collection (`[Type]`, `Array<Type>`, or `RuneString`) prior to constructing a mutable slice (`[ref Type..]`), detaching a unique deep copy via Copy-on-Write if `strongCount > 1`.

### Non-Escaping Restrictions

* MAY declare `[Type..]` and `[ref Type..]` as method parameters, local variables, or stage iteration sources.
* MUST NOT declare `[Type..]` or `[ref Type..]` as:
  * A structural or variant payload field of a `Data` type,
  * A `static`, `astatic`, or `gstatic` variable or field,
  * A method or anonymous function return type (`-> [Type..]`), or
  * A type argument to a generic `Data` type, collection, or `~Interface`.
* MUST NOT capture a `[Type..]` or `[ref Type..]` variable into a `spawn` or `rollback` block.
* MUST NOT reassign a local slice variable to reference a source collection declared in a narrower inner lexical scope than the slice variable itself.
