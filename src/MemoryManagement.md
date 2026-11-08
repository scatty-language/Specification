# Memory Management

## Value Semantics and Type Categories

### Trivial Types

* MUST classify a `Data` type as a Trivial Type if it consists entirely of primitives or other Trivial Types and does not implement `~Disposable`.
* MUST allocate Trivial Types as contiguous memory blocks directly on the stack or inline within an enclosing `Data`.
* MUST execute a bitwise copy upon assignment or pass-by-value of a Trivial Type.
* MUST guarantee complete logical and physical independence between the original and the copied variable.

```scatty
Vector(X: Int, Y: Int);

one := Vector { X: 10, Y: 10 };
two := one;
two.X = 20;

Console.WriteLine(one.X); // Output: 10
```

### Owning Types

* MUST classify a `Data` type as an Owning Type if it is a built-in heap-backed type (`String`, `RuneString`, dynamic or heap-allocated collections), implements the `~Disposable` interface, or contains any Owning Type or `weak` fields.
* MUST manage the underlying heap allocation of an Owning Type via an atomic ARC Control Block tracking a strong reference count and a weak reference count.
* MUST atomically increment the strong reference count upon assignment or pass-by-value of an Owning Type without duplicating the underlying heap memory.
* MUST atomically decrement the strong reference count of the previously held instance prior to overwriting an already initialized Owning Type variable.
* MUST NOT poison the source variable upon assignment or pass-by-value, both variables remain valid and accessible.

### Copy-on-Write (CoW)

* MUST evaluate the strong reference count prior to executing any mutating operation (`ref @` method or field reassignment) on an Owning Type instance.
* MUST execute the mutation directly in-place without allocating new memory if the instance is uniquely referenced (`strongCount == 1`).
* MUST automatically clone a deep copy of the underlying data, decrement the original instance's strong reference count, and apply the mutation to the newly allocated copy if the instance is shared (`strongCount > 1`).
* MUST ignore the weak reference count (`weakCount`) when determining uniqueness for Copy-on-Write. An instance with `strongCount == 1` and `weakCount > 0` MUST be mutated in-place so existing `weak` references remain attached to the mutated instance.

```scatty
list1 := [Int] { 1, 2, 3 };

// ARC retain: 'list1' and 'list2' temporarily share the same heap buffer (strongCount == 2)
list2 := list1;

// Mutation triggers Copy-on-Write because strongCount > 1
// 'list2' allocates its own copy before adding 4; 'list1' remains unchanged
list2.Add(4);

Console.WriteLine(list1.Count); // Output: 3
Console.WriteLine(list2.Count); // Output: 4
```

## Weak References

* MUST apply the `weak` type modifier exclusively to Owning Types.
* MUST NOT apply the `weak` type modifier to a Trivial Type or primitive.
* MUST atomically increment the target's weak reference count (`weakCount`) without incrementing its strong reference count (`strongCount`) when assigning an Owning Type to a `T weak` target.
* MUST allow the target's payload and `~Disposable` resources to be destroyed immediately when `strongCount` reaches `0`, even if `weakCount > 0`.
* MUST keep the ARC Control Block metadata allocated until both `strongCount` and `weakCount` reach `0`.
* MAY initialize or reassign a `T weak` variable or field with the `none` literal to represent an unattached weak reference (`strongCount == 0`) without allocating an ARC Control Block.

### Accessing and Unwrapping Weak References

* MUST NOT access members of a `T weak` expression directly via `.` or `[]`, or implicitly convert a `T weak` expression to a strong `T`.
* MUST explicitly unwrap or promote a `T weak` expression using conditional binding (`:=`), optional chaining (`?.` or `?[...]`), postfix force-unwrap (`!`), or the coalescing operator (`??`).
* MUST treat an unattached `none` weak reference identically to a disposed weak reference (`strongCount == 0`) during conditional binding (`must val := weakVar`), optional chaining (`?.` or `?[...]`), coalescing (`??`), and force-unwrapping (`!`).
* MAY apply the optional chaining (`?.` and `?[...]`), postfix force-unwrap (`!`), and coalescing (`??`) operators directly to a `T weak` expression:
  * `weakVar?.Member` atomically retains the target if `strongCount > 0` and evaluates to `MemberType?`, or yields `Option::None` without panicking if `strongCount == 0`.
  * `weakVar!` atomically promotes `weakVar` to a strong `T` if `strongCount > 0`, or immediately triggers runtime Abort or Fatal Panic mechanics if `strongCount == 0`.
  * `weakVar ?? fallback` atomically promotes `weakVar` to a strong `T` if `strongCount > 0`, or evaluates to `fallback` if `strongCount == 0`.

```scatty
Node : ~Copyable
{
    Value: Int;
    Parent: Node weak;
    Children: [Node];

    Copy(@) -> @ => Node { Value: @.Value, Parent: @.Parent, Children: @.Children };
}

parent := Node { Value: 1, Parent: none, Children: [Node].New() };
child := Node { Value: 2, Parent: parent, Children: [Node].New() };

// strongCount of 'parent' is 1; 'child.Parent' is weak and does not trigger CoW
parent.Children.Add(child);

// Safe conditional promotion to a strong reference
may strongParent := child.Parent
{
    Console.WriteLine(strongParent.Value); // Output: 1
}

// Explicit force-unwrap (promotes if alive, panics/aborts if disposed)
Console.WriteLine(child.Parent!.Value);

// ERROR: Cannot access members of a 'weak' reference directly without '!', '?.', or unwrapping
Console.WriteLine(child.Parent.Value);
```

## Borrowing and In-Place Mutation

* MAY pass any variable by reference using the `ref` keyword to mutate the caller's instance in-place without incrementing its ARC strong reference count.
* MUST explicitly declare the `ref` keyword at both the method parameter declaration and the invocation call site.
* MUST preserve `strongCount == 1` when an unshared variable is passed by `ref`, avoiding unnecessary Copy-on-Write allocations inside the callee.

```scatty
ClearList(ref list: [Int]) 
{
    // Mutates in-place without triggering CoW if 'myList' has strongCount == 1
    list.Clear();
}

myList := [Int] { 10, 20 };
ClearList(ref myList); 
```

### Law of Exclusivity

* MUST NOT alias a memory location passed as a `ref` argument with any other parameter (either by `ref` or by value) during the exact same method invocation.
* MAY pass distinct structural fields of the same `Data` instance simultaneously as separate `ref` arguments.
* MUST NOT pass a parent instance (by value or `ref`) and any of its nested fields simultaneously within the same method invocation.
* MUST treat an in-place indexed borrow (`ref col[key]` or invoking a `ref @` method on `col[key]`) as an exclusive `ref` borrow of the entire `col` collection for the duration of the method invocation.
* MUST NOT pass a collection (by value or `ref`) and any of its indexed elements simultaneously within the same method invocation.
* MUST NOT pass two indexed elements of the same collection (`ref col[i]` and `ref col[j]`) simultaneously as `ref` arguments within the same method invocation.
* MUST NOT pass a `static` or `astatic` variable or field directly as a `ref` argument to a method parameter.

```scatty
ProcessValues(ref a: Int, ref b: Int) { }
ProcessMixed(ref obj: Vector, ref val: Int) { }
ProcessListAndItem(ref list: [Int], ref item: Int) { }

vector := Vector { X: 0, Y: 0 };
numbers := [Int] { 10, 20, 30 };

// ERROR: 'vector' is passed by 'ref' and by value simultaneously
Process(ref vector, vector); 

// Valid: distinct non-overlapping fields
ProcessValues(ref vector.X, ref vector.Y);

// ERROR: 'vector' memory overlaps with 'vector.X'
ProcessMixed(ref vector, ref vector.X); 

// ERROR: cannot borrow 'numbers' and its element 'numbers[0]' simultaneously
ProcessListAndItem(ref numbers, ref numbers[0]);

// ERROR: cannot borrow two indexed elements of 'numbers' simultaneously
ProcessValues(ref numbers[0], ref numbers[1]);
```

## Resource Cleanup and Disposal

* MUST automatically decrement the strong reference count of all local Owning Type variables when they exit their lexical scope.
* MUST automatically destroy the instance and recursively decrement the strong reference counts of all nested owning fields precisely when the instance's `strongCount` reaches `0`.
* MUST NOT require manual implementation of cleanup interfaces solely to propagate ARC cleanup of nested fields.

### Custom Disposal and `~Copyable`

* MUST explicitly implement the `~Disposable` interface to manage raw unmanaged resources (e.g., OS file descriptors, unmanaged pointers) or execute custom side effects upon deterministic destruction.
* MUST execute the custom `Dispose(ref @)` method precisely when the instance's `strongCount` reaches `0`, prior to releasing any nested owning fields.
* MUST NOT directly invoke the `Dispose` method from safe user code.
* MUST also implement the `~Copyable` interface (`Copy(@) -> @`) on a custom `~Disposable` type if the type is mutated while shared (`strongCount > 1`). Otherwise, mutating a shared custom `~Disposable` type that does not implement `~Copyable` MUST be rejected or performed via unique ownership.
* MUST implicitly gather the scattered fields of each element into a temporary stack instance to execute `Dispose(ref temp)` when destroying or removing elements from a Structure of Arrays (`soa`) collection whose element type implements `~Disposable`.

```scatty
~Disposable 
{
    Dispose(ref @);
}

FileHandle : ~Disposable 
{
    _rawDescriptor: Int;
    Path: String;

    Dispose(ref @) => OS.Close(@._rawDescriptor);
}

ProcessFile() throws 
{
    file := FileHandle { _rawDescriptor: OS.Open("data.txt"), Path: "data.txt" };
    
    Parse(ref file); 

    // 'file' exits scope -> strongCount drops to 0:
    // 1. 'file.Dispose()' is executed
    // 2. 'file.Path' strongCount is decremented and deallocated
}
```

### Variant Payload Cleanup and Switching

* MUST inspect the active discriminant tag when a tagged `Data` instance exits its lexical scope or its `strongCount` reaches `0`, and decrement the strong reference count (invoking `Dispose` where applicable) for the Owning Type payload fields belonging to the currently active variant prior to releasing shared Owning Type fields.
* MUST execute an in-place variant switch on a tagged `Data` instance in the following strict three-step order to prevent use-after-free when the new variant's payload references data from the outgoing variant:
  1. Evaluate all payload argument expressions for the new variant into temporary bindings prior to mutating the target instance.
  2. Inspect the target instance's outgoing discriminant tag and release or dispose all Owning Type payload fields belonging to the outgoing active variant.
  3. Update the discriminant tag to the new variant and move the evaluated payload values from Step 1 into the variant payload storage.

```scatty
ConnectionState 
{
    ::Idle;
    ::Active(Handle: FileHandle, Log: String);
    ::Error(LastLog: String);

    Endpoint: String;
}

UpdateState() throws 
{
    file := FileHandle { _rawDescriptor: OS.Open("net.log"), Path: "net.log" };
    state := ConnectionState::Active(file, "Connected") { Endpoint: "localhost" };
    
    // Safe self-referential variant switch via '=::':
    // 1. Evaluates 'state.Log + " -> Failed"' while '::Active' payload is still alive
    // 2. Disposes '::Active' payload ('file.Dispose()' is called and 'state.Log' is released)
    // 3. Writes tag '::Error' and moves the new 'LastLog' string into the payload
    may state is .Active 
    {
        state =:: Error(state.Log + " -> Failed");
    }

    // 'state' exits scope:
    // 1. Releases active variant '::Error' payload ('LastLog')
    // 2. Releases shared field ('Endpoint')
}
```

### Associated Array Cleanup

* MUST decrement the strong reference count (and invoke `Dispose` where applicable) for all `VariantCount` elements when an associated array (`[K -> V]`) containing Owning Type elements (or `Option::Some` payloads in `[K -> V?]`) is destroyed.
