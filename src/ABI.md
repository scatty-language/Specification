# Application Binary Interface (ABI)

* MUST enforce a deterministic binary layout and calling convention across all compilation units within a target architecture.

## Data Layout and Alignment

* MUST align primitive types to their natural byte boundaries:

| Type | Size (bytes) | Alignment (bytes) |
| :--- | :--- | :--- |
| `Bool`, `Byte`, `Char` | 1 | 1 |
| `Int`, `Uint`, `Float`, `Rune` | 4 | 4 |
| `Long`, `Ulong`, `Double` | 8 | 8 |
| `Size`, `Usize` | 4 (32-bit) / 8 (64-bit) | 4 (32-bit) / 8 (64-bit) |

* MUST set the alignment of a `Data` type or tuple to the maximum alignment among its physical fields (minimum alignment of `1` byte).
* MUST round the total byte size of a `Data` type or tuple up to the nearest multiple of its alignment (minimum size of `1` byte).
* MUST NOT allocate bytes in the instance layout for computed properties, constants (`::`), or `static`, `astatic`, and `gstatic` fields.

### Default Optimized Layout

* MUST apply the default optimized layout to all tuples and all `Data` types that omit the `extern` modifier:
  1. Alignment Sorting: Physical fields MUST be sorted in descending order of their natural alignment requirement (`alignof`).
  2. Semantic Clustering: Within each group of fields sharing the exact same alignment requirement, the compiler MUST partition and order fields contiguously in the following sequence:
     * Fields declared with the `math` modifier, preserving their relative lexical declaration order.
     * Fields declared with the `equality` modifier (and lacking `math`), preserving their relative lexical declaration order.
     * All remaining unmarked fields, preserving their relative lexical declaration order.
  3. Tail-Padding Reuse: When a non-`extern` `Data` type contains a nested non-`extern` `Data` field whose layout includes trailing alignment padding bytes, the compiler MUST place subsequent smaller fields (or the enclosing type's discriminant tag) into those unused trailing padding bytes according to their alignment requirements prior to increasing the outer type's byte footprint.

### Sequential `extern` Layout

* MUST apply sequential C-compatible layout to any `Data` type declared with the `extern` modifier:
  1. Physical fields MUST be laid out sequentially in their exact lexical declaration order without reordering or semantic clustering.
  2. Each non-bitfield structural field MUST be aligned to the next memory offset that is a multiple of the field's natural alignment, inserting padding bytes between fields where required.
  3. Trailing padding bytes of nested `Data` fields MUST NOT be reused by fields of the enclosing `extern` `Data` type.
  4. Adjacent bitfields (`Field: Type bitwidth N;`) MUST be packed contiguously from the least significant bit to the most significant bit without intermediate byte padding until the underlying storage unit boundary is crossed.

## Tagged Unions and Niche Optimization

### Default Tagged `Data` Layout

* MUST lay out a tagged `Data` type (`Data` declaring variants without `tag none`) that omits the `extern` modifier as follows:
  1. Variant Payload Union: All declared variant payloads share a single contiguous memory union aligned to the maximum alignment of all declared variant payloads and sized to the maximum byte footprint of all declared variant payloads. Within each individual variant payload, multiple payload fields MUST be ordered using the Default Optimized Layout rules.
  2. Discriminant Tag Placement: If the Shared Fields or trailing padding of the Variant Payload Union contains an unused alignment padding hole across all variants whose size and alignment can accommodate the Discriminant Tag (`Int` by default, or `tag Type`), the compiler MUST place the Discriminant Tag inside the first such padding hole. Otherwise, the compiler MUST treat the Discriminant Tag and the Variant Payload Union as pseudo-fields and order them alongside the Shared Fields by descending alignment (`alignof`).

### Sequential `extern` Tagged `Data` Layout

* MUST lay out a tagged `Data` type declared with the `extern` modifier sequentially in three contiguous regions without overlapping or tail-padding reuse:
  1. Shared Fields: All physical instance fields declared outside the variants, placed sequentially in declaration order starting at byte `0`.
  2. Discriminant Tag: The active variant tag (`Int` by default, or the explicit scalar integer type specified via `tag Type`), aligned to the tag type's natural boundary immediately following the Shared Fields.
  3. Variant Payload Union: A single shared memory region aligned to the maximum alignment of all declared variant payloads and sized to the maximum byte footprint of all declared variant payloads, with payload fields inside each variant laid out sequentially in declaration order.

### Untagged Unions (`tag none`)

* MUST omit the Discriminant Tag (`0` bytes) when `tag none` is declared.

### Generalized Niche Optimization for `Option<T>` (`T?`)

* MUST apply niche optimization to `Option<T>` (`T?`), omitting the separate discriminant tag so that `sizeof T? == sizeof T` and `alignof T? == alignof T`, whenever `T` is a primitive, pointer, or non-`extern` `Data` type whose layout contains at least one field with an invalid bit pattern (niche) at any byte offset:
  1. Non-Null Pointer Niche: If `T` is an Owning Type (`~Disposable`), a function pointer (`fn`), or an inline non-`extern` `Data` type containing an Owning Type or `fn` field at any byte offset, `Option::None` (`none`) MUST be represented by storing `0x0` in the first such pointer slot.
  2. Boolean Niche: If `T` is `Bool` or contains a `Bool` field at any byte offset, `Option::None` MUST be represented by storing the byte value `0x02` in that `Bool` slot (`0x00` represents `false` and `0x01` represents `true`).
  3. Rune Niche: If `T` is `Rune` or contains a `Rune` field at any byte offset, `Option::None` MUST be represented by storing `0xFFFFFFFF` in that `Rune` slot.
  4. Discriminant Tag Niche: If `T` is a tagged `Data` type (not `tag none`) whose discriminant tag type has at least one unused representable integer value, `Option::None` MUST be represented by storing the numeric value `MaxVariantTag + 1` (or the lowest unused representable value of the tag type) in `T`'s discriminant tag slot.
* MUST select the first applicable niche in the priority order listed above (Pointer -> Boolean -> Rune -> Discriminant Tag) at the lowest byte offset within `T` when multiple niches exist.
* MUST NOT extract internal field niches from inside a `Data` type declared with the `extern` modifier.

## ARC Control Block and Owning Types

* MUST represent any Owning Type (`~Disposable`) variable, parameter, or field as a single pointer (`sizeof Usize`) referencing a contiguous heap allocation that prefixes the instance payload with an 8-byte ARC Control Block header:

| Offset (bytes) | Field | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | `strongCount` | `Uint` (32-bit atomic) | Active strong reference count |
| `4` | `weakCount` | `Uint` (32-bit atomic) | Active weak reference count |
| `8` | `payload` | `T` | Inline instance fields or collection header |

* MUST represent a `T weak` variable, parameter, or field where `T` is an Owning Type (`~Disposable`) as a single pointer (`sizeof Usize`) referencing the target's ARC Control Block allocation, or `0x0` when unattached (`none`).
* MUST lay out any `Data` type that does not implement `~Disposable` inline according to standard `Data` layout rules without an ARC Control Block header, even if it contains Owning Type or `weak` fields.
* MUST lay out a `T weak` variable, parameter, or field where `T` is a non `~Disposable` `Data` type inline with the exact same byte size and field offsets as `T`, representing each recursively nested Owning Type field as a pointer (`sizeof Usize`) to its respective ARC Control Block allocation (or `0x0` when initialized to `none`).

## Dynamic Dispatch and Virtual Method Tables

* MUST represent a polymorphic interface reference (`item: ~Interface`, `ref item: ~Interface`, or `catch e: ~Exception`) as a 2-word fat pointer (`2 * sizeof Usize`):

| Offset (bytes) | Field | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | `data_ptr` | `Usize` | Pointer to the concrete `Data` instance |
| `sizeof Usize` | `vtable_ptr` | `Usize` | Pointer to the static virtual method table (`vtable`) |

* MUST lay out the static virtual method table (`vtable`) for each `(ConcreteType, ~Interface)` pair as an array of pointer-sized words (`Usize`):

| Word Index | Entry | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | `size` | `Usize` | Byte size of `ConcreteType` |
| `1` | `align` | `Usize` | Byte alignment of `ConcreteType` |
| `2` | `dispose_fn` | `Usize` | Pointer to cleanup thunk `fn(ref ConcreteType)` (`0x0` for Trivial Types) |
| `3..N` | `member_slots` | `Usize` | Function pointers for object-safe interface methods and property getters in declaration order |

* MUST omit non-object-safe interface members (static methods, generic methods, and methods or properties referencing `@` outside the receiver position) from `member_slots`.
* MUST set `dispose_fn` to `0x0` if `ConcreteType` is a Trivial Type.
* MUST populate `dispose_fn` with a compiler-generated cleanup thunk if `ConcreteType` is an Owning Type (`~Disposable`) or an inline `Data` type containing nested Owning Type or `weak` fields; the thunk MUST decrement `strongCount` (or `weakCount`) and invoke `Dispose` where applicable for the instance pointed to by `data_ptr`.
* MUST populate the corresponding `member_slots` entry with `0x0` if a weak interface contract member (`Method?` or `Property?`) is omitted by `ConcreteType`.

## Calling Convention

* MUST compile all methods (omitting `extern`), virtual table thunks, and stateless function pointers (`fn` omitting `extern`) using the LLVM `fastcc` register-prioritized calling convention.
* MUST compile all methods declared with `extern` and function pointers declared with `extern` (`fn(...) extern`) using the target platform's native C ABI calling convention (`ccc`).
* MUST disable interprocedural register-allocation, parameter-reordering, and dead-argument-elimination optimizations on methods and `fn` pointers declared with `extern`.
* MUST represent both standard `fn` and `fn(...) extern` pointers as a single raw code address (`sizeof Usize`) with zero closure environment overhead.
* MUST map the instance receiver (`@` or `ref @`) to the first parameter slot (index `0`), guaranteeing binary ABI compatibility between instance methods and stateless `fn` pointers whose first parameter matches the receiver type and `extern` qualification.

### Parameter Passing and Return Values

* For non-`extern` (`fastcc`) methods and `fn` pointers:
  * MUST pass non-`ref` parameters (including immutable `@` receivers) whose byte size is less than or equal to `sizeof Usize` directly by value in CPU registers.
  * MUST pass non-`ref` parameters (including immutable `@` receivers) whose byte size exceeds `sizeof Usize` via a hidden read-only pointer in a CPU register, pointing to the caller's source instance when unaliased or to a caller-allocated temporary stack copy when rvalue or call-site value materialization is required.
  * MUST return values whose byte size is less than or equal to `sizeof Usize` directly in CPU return registers, and values exceeding `sizeof Usize` via Return Value Optimization (RVO) using a hidden structure-return (`sret`) pointer register.
* For `extern` (`ccc`) methods and `fn(...) extern` pointers:
  * MUST pass by-value parameters and return values according to the target architecture's native C ABI register classification and stack-passing rules (using caller-allocated temporary copies when call-site value materialization is required).
  * MUST pass `ref` parameters and `ref @` receivers as a direct memory address (`Usize`) in the corresponding C ABI pointer register or stack slot.
  * MUST pass polymorphic interface parameters (`~Interface`) and slices (`[T..]`, `[ref T..]`) as 2-word structures according to the target platform's native C ABI rules for 2-word aggregates.

### Return Values

* MUST NOT allocate a return register for methods that omit the `-> ReturnType` clause.
* MUST return values whose byte size is less than or equal to `sizeof Usize` directly in CPU return registers.
* MUST return values whose byte size exceeds `sizeof Usize` via Return Value Optimization (RVO) using a hidden structure-return (`sret`) pointer register pointing to caller-allocated stack memory.

## Exception Propagation ABI

* MUST allocate a 16-byte-aligned, 128-byte exception buffer in Thread-Local Storage (TLS) for each thread.
* MUST reserve a dedicated error indicator register (`swifterror` / error slot pointer register) across all methods and `fn` pointers declared with the `throws` modifier:
  * Upon a non-throwing return, the callee MUST set the error indicator register to `0x0` and populate the normal return register or `sret` RVO buffer.
  * Upon executing `throw`, the callee MUST write the exception value bitwise into the 128-byte TLS exception buffer, set the error indicator register to the non-null address of the TLS exception buffer, execute frame-local `undo` and ARC / `~Disposable` cleanup, and return via a standard `ret` instruction while leaving the primary return register or `sret` RVO buffer uninitialized (`noinit`).

## Static Storage and Spinlock ABI

* MUST allocate `static` fields and local variables in Thread-Local Storage (TLS) aligned to the type's natural alignment without synchronization headers.
* MUST allocate `gstatic` fields and local variables in the process-wide global data segment aligned to the type's natural alignment without synchronization headers.
* MUST allocate lock-free `astatic` fields and local variables (Trivial Types whose byte size is `8` bytes or less) in the process-wide global data segment aligned to the smallest power-of-two byte boundary greater than or equal to the type's byte size (`1`, `2`, `4`, or `8` bytes).
* MUST compile reads, writes, and read-modify-write operations on lock-free `astatic` storage as single hardware atomic instructions using sequentially consistent (`seq_cst`) memory ordering.
* MUST allocate spinlock-backed `astatic` fields and local variables (Owning Types, types containing nested Owning Type or `weak` fields, or Trivial Types exceeding `8` bytes) in the process-wide global data segment with a preceding atomic spinlock header:

| Offset (bytes) | Field | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | `owner_thread` | `Uint` (32-bit atomic) | `0` when unlocked, non-zero owning thread ID when locked |
| `align_up(4, alignof T)` | `payload` | `T` | Inline instance fields or Owning Type pointer |

* MUST assign each active thread a unique, non-zero 32-bit thread identifier (`tid`).
* MUST execute spinlock acquisition on `owner_thread` via an atomic compare-and-exchange (`cmpxchg`) transitioning from `0` to the calling thread's `tid` with `acquire` memory ordering.
* MUST inspect the existing `owner_thread` value upon a failed `cmpxchg` attempt and immediately trigger runtime Abort or Fatal Panic mechanics if `owner_thread` equals the calling thread's `tid`.
* MUST execute spinlock release by atomically storing `0` into `owner_thread` with `release` memory ordering.
* MUST emit spinlock release instructions on both normal return and non-null error indicator register (`swifterror`) paths immediately following a guarded `throws` invocation, prior to executing any deferred ARC reference decrements, `Dispose` calls, or exception propagation branches.

## Slices ABI

* MUST represent a slice (`[T..]` or `[ref T..]`) as a 2-word structure (`2 * sizeof Usize`) with alignment equal to `alignof Usize`:

| Offset (bytes) | Field | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | `data_ptr` | `Usize` | Pointer to the first contiguous `T` element (`0x0` when empty) |
| `sizeof Usize` | `length` | `Usize` | Element count (zero-extended from `Int`) |

* MUST compile a slice parameter (`local_id: T...`) with the exact same binary parameter signature as `[T..]`, guaranteeing binary ABI compatibility between a method declaring `T...` and a function pointer `fn(..., [T..])`.
