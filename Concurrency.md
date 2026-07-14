# Concurrency

Coggy strictly utilizes Structured Concurrency. Concurrency is lexically scoped, guaranteeing deterministic execution trees, strict lifetime bounds, and data-race freedom at compile time.

## Concurrent Scopes and Spawn

The `concurrent` keyword introduces an asynchronous execution boundary. Within this boundary, the `spawn` keyword is used to dispatch concurrent tasks.

```EBNF
ConcurrentStmt ::= "concurrent" Block
SpawnStmt      ::= "spawn" (Block | "=>" Statement)
```

* **Implicit Join**: A `concurrent` block MUST NOT exit until all logically spawned tasks within its lexical scope have strictly completed. The compiler automatically inserts a synchronization barrier at the end of the `concurrent` block.
* **No Escaping**: The `spawn` keyword MUST ONLY be used within the lexical scope of a `concurrent` block.

```C#
ProcessData(data: int) { }

// Execution blocks here until all spawned tasks complete
concurrent
{
    spawn => ProcessData(1);
    spawn => ProcessData(2);
    
    spawn 
    {
        ProcessData(3);
        ProcessData(4);
    }
}
```

## Capture and Memory Safety Rules

When a variable from an outer scope is accessed inside a `spawn` block, strict capture rules apply to prevent data races:

1. **Moves**: Variables passed by value or via Sharp Pointers are moved into the task. The task assumes full ownership.
2. **Immutable View Pointers**: Tasks MAY capture immutable view pointers. This is strictly safe because the `concurrent` barrier guarantees the task will terminate before the Last Use Place (LUP) of the data. Multiple spawned tasks MAY capture the same `&` pointer concurrently.
3. **Mutable View Pointers**: STRICTLY FORBIDDEN. A `spawn` block MUST NOT capture a mutable view pointer from an outer scope. This statically eliminates data races without requiring a complex runtime borrow checker.

```C#
Calculate(view: int&) { }

mut counter := 0;
dataPtr: int# = 10#;
sharedView: int& = &dataPtr;

concurrent 
{
    // Valid: Immutable view pointer is shared across tasks safely.
    spawn => Calculate(sharedView);
    spawn => Calculate(sharedView);
    
    // Error: Cannot capture mutable variable in a concurrent task
    spawn => counter++;
}
```

## Returning Values from Tasks

Spawned tasks MAY return values. The `concurrent` block itself can evaluate to a collection of these returned values, implicitly acting as an aggregator.

* If a `spawn` block yields a value, the enclosing `concurrent` block evaluates to an array of those values.
* All `spawn` branches yielding a value MUST return the exact same type.

```C#
// Returns an array of integers: int[]
results := concurrent 
{
    spawn => 10;
    spawn => 20;
    spawn => 30;
};
```

## Exceptions in Structured Concurrency

Structured Concurrency inherently encapsulates error states. If an unhandled exception escapes a `spawn` block, it fundamentally changes the state of the enclosing `concurrent` scope.

* **Fail-Fast by Default**: Because `concurrent` blocks are non-throwing by default, if any spawned task throws an exception, the entire program aborts immediately.
* **Cooperative Cancellation (`concurrent throws`)**: If the block is marked with `throws`:
  1. If ANY task throws, the `concurrent` block transitions to a `Cancelled` state.
  2. Other tasks MUST cooperatively observe cancellation.
  3. The `concurrent` block waits for all tasks to acknowledge cancellation, then strictly rethrows an `#AggregateException`.

### The Rollback Sub-Block

To facilitate transactional semantics without complex manual compensations, `spawn` supports an attached `rollback` block.

```EBNF
SpawnStmt ::= "spawn" (Block | "=>" Statement) ["rollback" (Block | "=>" Statement)]
```

* **Execution Condition**: The `rollback` block strictly executes ONLY IF its associated `spawn` task has strictly and successfully completed, AND the enclosing `concurrent` scope subsequently enters a `Cancelled` state (due to an exception in another task).
* **Variable Capture Rules**: Variable access inside a `rollback` block follows standard affine capture rules.
* **Implicit Non-Throwing**: The `rollback` block is MUST non-throwing. If an exception is thrown inside a `rollback` block, the program MUST immediately abort.

```C#
concurrent throws 
{
    // Task 1: Uploads a file
    spawn 
    {
        fileId := Storage.Upload(fileData);
    } 
    rollback 
    {
        // Executes if Task 1 succeeds, but Task 2 fails
        // ABORTS if Delete throws (rollback is strictly non-throwing)
        Storage.Delete(fileId);
    }

    // Task 2: Fails and triggers cancellation
    spawn 
    {
        throw #Exception.New("Database disconnected");
    }
}
```
