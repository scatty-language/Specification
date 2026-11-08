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
* MUST apply pointer niche optimization to `Option<T>` (`T?`) whenever `T` is an Owning Type or a function pointer (`fn`):
  * `Option<T>` MUST occupy exactly `sizeof(Usize)` bytes with no separate discriminant tag.
  * `Option::None` (`none`) MUST be represented as a null pointer address (`0x0`).
  * `Option::Some(val)` MUST be represented directly as the non-null pointer address of `val`.

## ARC Control Block and Owning Types

* MUST represent any Owning Type variable, parameter, or field as a single pointer (`sizeof(Usize)`) referencing a contiguous heap allocation that prefixes the instance payload with an 8-byte ARC Control Block header:

| Offset (bytes) | Field | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | `strongCount` | `Uint` (32-bit atomic) | Active strong reference count |
| `4` | `weakCount` | `Uint` (32-bit atomic) | Active weak reference count |
| `8` | `payload` | `T` | Inline instance fields or collection header |

* MUST represent a `T weak` variable, parameter, or field as a single pointer (`sizeof(Usize)`) referencing the target's ARC Control Block allocation, or `0x0` when unattached (`none`).

## Dynamic Dispatch and Virtual Method Tables

* MUST represent a polymorphic interface reference (`ref item: ~Interface`, `Box<~Interface>`, or `catch e: ~Exception`) as a 2-word fat pointer (`2 * sizeof(Usize)`):

| Offset (bytes) | Field | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | `data_ptr` | `Usize` | Pointer to the concrete `Data` instance |
| `sizeof(Usize)` | `vtable_ptr` | `Usize` | Pointer to the static virtual method table (`vtable`) |

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
* MUST represent a function pointer (`fn`) as a single raw code address (`sizeof(Usize)`) with zero closure environment overhead.
* MUST map the instance receiver (`@` or `ref @`) to the first parameter slot (index `0`), guaranteeing binary ABI compatibility between instance methods and stateless `fn` pointers whose first parameter matches the receiver type.

### Parameter Passing

* MUST pass non-`ref` parameters (including immutable `@` receivers) whose byte size is less than or equal to `sizeof(Usize)` directly by value in CPU registers.
* MUST pass non-`ref` parameters (including immutable `@` receivers) whose byte size exceeds `sizeof(Usize)` via a hidden read-only pointer in a CPU register.
* MUST pass `ref` parameters and `ref @` receivers as a direct memory address (`Usize`) in a CPU register.

### Return Values

* MUST NOT allocate a return register for methods that omit the `-> ReturnType` clause.
* MUST return values whose byte size is less than or equal to `sizeof(Usize)` directly in CPU return registers.
* MUST return values whose byte size exceeds `sizeof(Usize)` via Return Value Optimization (RVO) using a hidden structure-return (`sret`) pointer register pointing to caller-allocated stack memory.

## Exception Propagation ABI

* MUST allocate a 16-byte-aligned, 128-byte exception buffer in Thread-Local Storage (TLS) for each thread.
* MUST reserve a dedicated error indicator register (`swifterror` / error slot pointer register) across all methods and `fn` pointers declared with the `throws` modifier:
  * Upon a non-throwing return, the callee MUST set the error indicator register to `0x0` and populate the normal return register or `sret` RVO buffer.
  * Upon executing `throw`, the callee MUST write the exception value bitwise into the 128-byte TLS exception buffer, set the error indicator register to the non-null address of the TLS exception buffer, execute frame-local `undo` and ARC / `~Disposable` cleanup, and return via a standard `ret` instruction while leaving the primary return register or `sret` RVO buffer uninitialized (`noinit`).
