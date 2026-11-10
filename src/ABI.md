# Application Binary Interface (ABI)

* MUST enforce a deterministic binary layout and calling convention across all compilation units within a target architecture.

## Data Layout and Alignment

* MUST lay out physical fields of `Data` types and tuples sequentially in their exact declaration order.
* MUST NOT reorder fields automatically during compilation.
* MUST align primitive types to their natural byte boundaries:

| Type | Size (bytes) | Alignment (bytes) |
| :--- | :--- | :--- |
| `Bool`, `Byte`, `Char` | 1 | 1 |
| `Int`, `Uint`, `Float`, `Rune` | 4 | 4 |
| `Long`, `Ulong`, `Double` | 8 | 8 |
| `Size`, `Usize` | 4 (32-bit) / 8 (64-bit) | 4 (32-bit) / 8 (64-bit) |

* MUST align each structural field to a memory offset that is a multiple of the field's natural alignment, inserting padding bytes between fields where required.
* MUST set the alignment of a `Data` type or tuple to the maximum alignment among its physical fields (minimum alignment of `1` byte).
* MUST round the total byte size of a `Data` type or tuple up to the nearest multiple of its alignment (minimum size of `1` byte).
* MUST pack adjacent bitfields (`Field: Type bitwidth N;`) contiguously from the least significant bit to the most significant bit without intermediate byte padding until the underlying storage unit boundary is crossed.
* MUST NOT allocate bytes in the instance layout for computed properties, constants (`::`), or `static`, `astatic`, and `gstatic` fields.

## Tagged Unions and Niche Optimization

* MUST lay out a tagged `Data` type (`Data` declaring variants without `tag none`) sequentially in three contiguous regions:
  1. Shared Fields: All physical instance fields declared outside the variants, placed at fixed offsets starting at byte `0`.
  2. Discriminant Tag: The active variant tag (`Int` by default, or the explicit scalar integer type specified via `tag Type`), aligned to the tag type's natural boundary.
  3. Variant Payload Union: A single shared memory region aligned to the maximum alignment of all declared variant payloads and sized to the maximum byte footprint of all declared variant payloads.
* MUST omit the Discriminant Tag region (`0` bytes) when `tag none` is declared, placing the Variant Payload Union immediately after the Shared Fields region.
* MUST apply pointer niche optimization to `Option<T>` (`T?`) whenever `T` is an Owning Type, a function pointer (`fn`), or an inline `Data` type containing an Owning Type field at offset `0`:
  * `Option::None` (`none`) MUST be represented by setting the pointer slot at offset `0` to `0x0` with no separate discriminant tag.
  * `Option::Some(val)` MUST be represented directly by the non-null pointer address in the slot at offset `0`.

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

* MUST represent a polymorphic interface reference (`ref item: ~Interface`, `Box<~Interface>`, or `catch e: ~Exception`) as a 2-word fat pointer (`2 * sizeof Usize`):

| Offset (bytes) | Field | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | `data_ptr` | `Usize` | Pointer to the concrete `Data` instance |
| `sizeof Usize` | `vtable_ptr` | `Usize` | Pointer to the static virtual method table (`vtable`) |

* MUST lay out the static virtual method table (`vtable`) for each `(ConcreteType, ~Interface)` pair as an array of pointer-sized words (`Usize`):

| Word Index | Entry | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | `size` | `Usize` | Byte size of `ConcreteType` |
| `1` | `align` | `Usize` | Byte alignment of `ConcreteType` |
| `2` | `dispose_fn` | `Usize` | Cleanup function pointer (`0x0` for Trivial Types) |
| `3..N` | `member_slots` | `Usize` | Function pointers for interface methods and property getters in declaration order |

* MUST populate the corresponding `member_slots` entry with `0x0` if a weak interface contract member (`Method?` or `Property?`) is omitted by `ConcreteType`.

## Calling Convention

* MUST compile all methods, virtual table thunks, and stateless function pointers (`fn`) using the LLVM `fastcc` register-prioritized calling convention.
* MUST represent a function pointer (`fn`) as a single raw code address (`sizeof Usize`) with zero closure environment overhead.
* MUST map the instance receiver (`@` or `ref @`) to the first parameter slot (index `0`), guaranteeing binary ABI compatibility between instance methods and stateless `fn` pointers whose first parameter matches the receiver type.

### Parameter Passing

* MUST pass non-`ref` parameters (including immutable `@` receivers) whose byte size is less than or equal to `sizeof Usize` directly by value in CPU registers.
* MUST pass non-`ref` parameters (including immutable `@` receivers) whose byte size exceeds `sizeof Usize` via a hidden read-only pointer in a CPU register.
* MUST point the hidden read-only pointer directly to the caller's source instance when the argument is an unaliased local binding, or to a caller-allocated temporary stack copy when the argument is an rvalue or requires call-site value materialization.
* MUST pass `ref` parameters and `ref @` receivers as a direct memory address (`Usize`) in a CPU register.

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
