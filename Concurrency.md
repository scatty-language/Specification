# Concurrency

## Concurrent Scopes and Spawn

* MUST restrict the usage of the `spawn` keyword to the lexical scope of a `concurrent` block.
* MUST NOT exit a `concurrent` block until all tasks spawned within its lexical scope have completed.
* MUST synchronize and join all active child tasks at the closing boundary of the `concurrent` block.
* MAY delegate scheduling and execution resilience for a task using the `by` modifier on a `spawn` statement.
* MUST resolve the expression supplied to the `by` modifier to a type implementing the `~SpawnStrategy` interface.

```C#
ProcessData(data: Int) { }

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

## Variable Capture and Memory Safety

* MUST capture variables referenced inside a `spawn` block by value.
* MUST NOT capture outer variables by reference (`ref`) inside a `spawn` block.
* MUST perform a bitwise copy when capturing Trivial Types into a `spawn` block.
* MUST atomically increment the ARC strong reference count (`strongCount`) when capturing Owning Types into a `spawn` block.
* MUST NOT invalidate or poison captured identifiers in the outer lexical scope or sibling `spawn` blocks.
* MUST evaluate mutations to captured Owning Types according to Copy-on-Write (CoW) semantics, cloning an isolated task-local copy prior to mutation if `strongCount > 1`.
* MUST NOT capture an Owning Type that implements `~Disposable` without implementing `~Copyable` across multiple `spawn` blocks, unless encapsulated in a thread-safe synchronization primitive.

```C#
Calculate(value: Int) { }

concurrent 
{
    counter := 0;
    data := [+ String].New();
    data.Add("Initial");

    spawn 
    {
        Calculate(counter);
        data.Add("Processed by Task 1");
    }
    
    spawn 
    {
        Calculate(counter);
        data.Add("Processed by Task 2"); 
    }
}
```

## Task Value Aggregation

* MAY evaluate a `concurrent` block as an expression yielding an array of values if its enclosed `spawn` statements return values.
* MUST evaluate all value-yielding `spawn` statements within the same `concurrent` block to the exact same type.
* MUST collect and return the evaluated results of all spawned tasks as an array.

```C#
results: [Int] = concurrent 
{
    spawn => 10;
    spawn => 20;
    spawn => 30;
};
```

## Error Handling and Cancellation

* MUST immediately abort the process if an unhandled exception escapes a `spawn` block inside a non-throwing `concurrent` block.
* MAY attach one or more `catch` arms to a `spawn` statement.
* MUST suppress exception propagation and prevent scope cancellation if a thrown exception is caught by a `catch` arm attached to the `spawn` statement.
* MAY declare a `concurrent` block with the `throws` modifier.
* MUST transition a `concurrent throws` block into a `Cancelled` state immediately upon any unhandled exception escaping a child task.
* MUST cooperatively signal cancellation to all sibling tasks upon entering the `Cancelled` state.
* MUST await completion of all tasks in the scope (including execution of applicable `rollback` blocks) and propagate the first unhandled exception that triggered cancellation through the calling thread's TLS exception buffer when a `concurrent throws` block is cancelled.
* MUST release and discard any subsequent unhandled exceptions thrown by sibling tasks after the `concurrent throws` scope has already entered the `Cancelled` state.

## Rollback Blocks

* MAY attach a `rollback` block to a `spawn` statement.
* MUST execute a `rollback` block if and only if its associated `spawn` task completed successfully and the enclosing `concurrent` scope subsequently entered a `Cancelled` state.
* MUST capture variables inside a `rollback` block by value.
* MUST NOT allow exceptions to escape from a `rollback` block.
* MUST immediately abort the process if an exception escapes from a `rollback` block.
* MAY declare a resilience strategy for a `rollback` block using the `by` modifier.
* MUST resolve the expression supplied to a `rollback by` modifier to a type implementing the `~SpawnStrategy` interface.

```C#
use System::Concurrency;

concurrent throws 
{
    spawn by Retry.New(3).WithDelay(100) 
    {
        fileId := Storage.Upload(fileData);
    } 
    catch e: ~Exception 
    {
        Logger.Warn("Upload failed completely");
    }
    rollback by Retry.New(3)
    {
        Storage.Delete(fileId);
    }

    spawn 
    {
        throw Exception.New("Database disconnected");
    }
}
```
