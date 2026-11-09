# Memory Management

## Value Semantics and Type Categories

### Trivial Types

* MUST classify a `Data` type as a Trivial Type if it does not implement `~Disposable` and contains no Owning Type or `weak` fields.
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

* MUST classify a `Data` type as an Owning Type if and only if it implements the `~Disposable` interface.
* MUST manage the underlying heap allocation of an Owning Type via an atomic ARC Control Block tracking a strong reference count (`strongCount`) and a weak reference count (`weakCount`).
* MUST atomically increment `strongCount` upon assignment or pass-by-value of an Owning Type without duplicating the underlying heap memory.
* MUST atomically decrement `strongCount` of the previously held instance prior to overwriting an already initialized Owning Type variable.
* MUST NOT poison the source variable upon assignment or pass-by-value; both variables remain valid and accessible.
* MUST allocate a non-`~Disposable` `Data` type that contains Owning Type or `weak` fields inline on the stack or within an enclosing `Data` layout without its own ARC Control Block.
* MUST copy trivial fields bitwise and atomically increment `strongCount` (or `weakCount`) of each nested Owning Type (or `weak`) field individually upon assignment or pass-by-value of a non-`~Disposable` `Data` type.

### Copy-on-Write

* MUST evaluate the strong reference count prior to executing any mutating operation (`ref @` method or field reassignment) on an Owning Type instance.
* MUST execute the mutation directly in-place without allocating new memory if the instance is uniquely referenced (`strongCount == 1`).
* MUST automatically clone a deep copy of the underlying data, decrement the original instance's strong reference count, and apply the mutation to the newly allocated copy if the instance is shared (`strongCount > 1`).
* MUST ignore the weak reference count (`weakCount`) when determining uniqueness for Copy-on-Write. An instance with `strongCount == 1` and `weakCount > 0` MUST be mutated in-place so existing `weak` references remain attached to the mutated instance.
* MUST evaluate Copy-on-Write exclusively on the individual mutated Owning Type field when mutating a non-`~Disposable` `Data` instance that contains Owning Type fields.

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

* MAY apply the `weak` type modifier to any Owning Type (`~Disposable`) or any `Data` type that recursively contains at least one Owning Type field.
* MUST NOT apply the `weak` type modifier to scalar primitives or Trivial Types that contain no Owning Type fields.
* MUST reject at compile time a direct inline self-referential `weak` field (`Field: Self weak`) on a `Data` type that does not implement `~Disposable`.
* MUST recursively transform all nested Owning Type fields within `T` into weak references when the `weak` modifier is applied to a non-`~Disposable` `Data` type (`T weak`).
* MUST atomically increment the weak reference count (`weakCount`) of the target Owning Type (or of every nested Owning Type field within an inline `Data` type) without incrementing `strongCount` when assigning an instance to a `T weak` target.
* MUST allow an Owning Type's payload and `~Disposable` resources to be destroyed immediately when its `strongCount` reaches `0`, even if `weakCount > 0`.
* MUST keep an Owning Type's ARC Control Block metadata allocated until both `strongCount` and `weakCount` reach `0`.
* MAY initialize or reassign a `T weak` variable or field with the `none` literal to represent an unattached weak reference (`strongCount == 0`) without allocating an ARC Control Block.

### Accessing and Unwrapping Weak References

* MUST NOT access members of a `T weak` expression directly via `.` or `[]`, or implicitly convert a `T weak` expression to a strong `T`.
* MUST explicitly unwrap or promote a `T weak` expression using conditional binding (`:=`), optional chaining (`?.` or `?[...]`), postfix force-unwrap (`!`), or the coalescing operator (`??`).
* MUST evaluate a `T weak` instance as live if and only if the target Owning Type (or every recursively nested Owning Type field within an inline `Data` type) has `strongCount > 0`.
* MUST atomically increment `strongCount` for all underlying Owning Type references upon promoting a live `T weak` instance to a strong `T`, or release any partially retained references and evaluate as disposed if any nested Owning Type field has `strongCount == 0`.
* MUST treat an unattached `none` weak reference identically to a disposed weak reference (`strongCount == 0`) during conditional binding (`must val := weakVar`), optional chaining (`?.` or `?[...]`), coalescing (`??`), and force-unwrapping (`!`).
* MAY apply the optional chaining (`?.` and `?[...]`), postfix force-unwrap (`!`), and coalescing (`??`) operators directly to a `T weak` expression:
  * `weakVar?.Member` atomically promotes the target to a strong `T` if live and evaluates to `MemberType?`, or yields `Option::None` without panicking if disposed (`strongCount == 0`).
  * `weakVar!` atomically promotes `weakVar` to a strong `T` if live, or immediately triggers runtime Abort or Fatal Panic mechanics if disposed (`strongCount == 0`).
  * `weakVar ?? fallback` atomically promotes `weakVar` to a strong `T` if live, or evaluates to `fallback` if disposed (`strongCount == 0`).

```scatty
Node : ~Disposable, ~Copyable
{
    Value: Int;
    Parent: Node weak;
    Children: [Node];

    Dispose(ref @) { }
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

* MUST treat an in-place method receiver (`ref @`) as an exclusive `ref` borrow of the receiver instance for the duration of the method invocation.
* MUST NOT alias or overlap a memory location bound to a `ref` parameter or `ref @` receiver with any other `ref` argument during the exact same method invocation.
* MAY pass distinct, non-overlapping structural fields of the same `Data` instance simultaneously as separate `ref` arguments, provided the parent instance itself is not bound to a `ref` parameter or `ref @` receiver in that invocation.
* MUST NOT pass a parent instance and any of its nested fields simultaneously as `ref` arguments (or as `ref @` and a `ref` argument) within the same method invocation.
* MUST treat an in-place indexed borrow (`ref col[key]` or invoking a `ref @` method on `col[key]`) as an exclusive `ref` borrow of the entire `col` collection for the duration of the method invocation.
* MUST NOT pass a collection and any of its indexed elements, or two indexed elements of the same collection (`ref col[i]` and `ref col[j]`), simultaneously as `ref` arguments (or as `ref @` and a `ref` argument) within the same method invocation.
* MUST NOT pass the exact same variable path simultaneously as an explicit `ref` argument and as a by-value argument (`Process(ref vector, vector)`).
* MUST materialize an isolated temporary copy at the call site prior to invocation whenever a by-value argument expression reads from a field, property, or indexed element of a root variable that is also bound to a `ref @` receiver or `ref` parameter in the same invocation.
* MUST NOT pass a `static` or `astatic` variable or field (or any sub-expression rooted in one) as a `ref` argument to a method parameter.

```scatty
Process(ref a: Vector, b: Vector) { }
ProcessValues(ref a: Int, ref b: Int) { }
ProcessMixed(ref obj: Vector, ref val: Int) { }
ProcessListAndItem(ref list: [Int], ref item: Int) { }

vector := Vector { X: 10, Y: 20 };
numbers := [Int] { 10, 20, 30 };

// ERROR: 'vector' is passed as an explicit 'ref' argument and by value simultaneously
Process(ref vector, vector); 

// Valid: distinct non-overlapping fields passed by 'ref'
ProcessValues(ref vector.X, ref vector.Y);

// ERROR: 'ref vector' memory overlaps with 'ref vector.X'
ProcessMixed(ref vector, ref vector.X); 

// ERROR: cannot borrow 'numbers' and its element 'numbers[0]' by 'ref' simultaneously
ProcessListAndItem(ref numbers, ref numbers[0]);

// ERROR: cannot borrow two indexed elements of 'numbers' by 'ref' simultaneously
ProcessValues(ref numbers[0], ref numbers[1]);

// Valid: 'vector.X' and 'numbers[0]' are passed by value
// compiler materializes isolated temporary copies prior to the 'ref @' mutation
vector.Translate(vector.X, 5);

numbers.Add(numbers[0]);
```

## Resource Cleanup and Disposal

* MUST automatically decrement `strongCount` of all local Owning Type variables and all nested Owning Type fields of inline `Data` variables when they exit their lexical scope.
* MUST automatically destroy an Owning Type instance and recursively decrement `strongCount` of all its nested Owning Type fields precisely when its `strongCount` reaches `0`.
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
