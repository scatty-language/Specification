# Copy-Update

* MUST perform a structural bitwise copy and apply mutations for Trivial Types.
* MUST copy trivial fields bitwise, retain unmodified nested Owning Type fields via ARC, and assign the newly evaluated expressions to the specified fields when evaluating a `with` expression on a non-`~Disposable` `Data` type.
* MUST preserve the original variable as valid and accessible when evaluating a `with` expression on an Owning Type (`~Disposable`) by allocating a new heap instance, copying trivial fields bitwise, retaining unmodified nested Owning Type fields via ARC, and applying the `CopyUpdateBody` assignments.
* MAY mutate the instance in-place as a compiler optimization if the target expression of `with` is an rvalue or a uniquely referenced Owning Type (`strongCount == 1`).
* MUST overwrite corresponding fields in the resulting instance with explicitly declared assignments in the `CopyUpdateBody`.
* MUST resolve the assigned expression to the exact type of the target field.
* MUST retain the original instance's value for any field omitted from the `CopyUpdateBody`.
* MUST NOT update restricted (private or internal) fields from outside their permitted lexical scope.
* MUST update physical fields only.
* MUST NOT directly update transparently forwarded members of an Embedded Field.
* MUST assign a new instance to the root identifier of an Embedded Field to mutate its state via `with`.

```scatty
ServerConfig 
{
    Host: String;
    Port: Int;
    Timeout: Int;
}

defaultConfig := ServerConfig 
{ 
    Host: "localhost", 
    Port: 8080, 
    Timeout: 3000 
};

// 'defaultConfig' remains valid; 'Host' buffer is shared via ARC
testConfig := defaultConfig with 
{ 
    Port: 9000, 
    Timeout: 5000 
};

Console.WriteLine(defaultConfig.Port); // Output: 8080
Console.WriteLine(testConfig.Port);    // Output: 9000
```

## Variant Copy-Update

* MAY switch the active variant of a tagged `Data` instance when constructing a modified copy via `with` (`instance with ::Variant` or `instance with ::Variant(args)`).
* MAY combine a variant switch and a `CopyUpdateBody` in a single `with` expression (`instance with ::Variant(args) { Field: value }`).
* MUST copy and preserve all shared fields from the source instance that are not explicitly overwritten in the `CopyUpdateBody`.
* MUST release or dispose any Owning Type payload fields (and nested Owning Type fields of inline payload types) belonging to the outgoing active variant if the `with` expression is optimized to mutate an rvalue or uniquely referenced Owning Type instance (`strongCount == 1`) in-place.
* MUST statically type the result of `instance with ::Variant(...)` as the narrowed variant subtype `Data::Variant` (which implicitly downgrades to the base `Data` type when bound via `:=`).
* MUST NOT apply a variant update (`with ::Variant`) to an untagged union (`tag none`).

```scatty
Bill 
{
    Id: Int;
    Amount: Float;

    ::Init;
    ::Closed(TransactionId: Int);

    Close(@::Init, txId: Int) -> @::Closed 
    {
        // Preserves '@.Id' and '@.Amount' while switching variant to '::Closed(txId)'
        return @ with ::Closed(txId);
    }

    CloseWithDiscount(@::Init, txId: Int, newAmount: Float) -> @::Closed 
    {
        // Switches variant to '::Closed(txId)' and updates shared field 'Amount'
        return @ with ::Closed(txId) { Amount: newAmount };
    }
}

initBill: Bill::Init = Bill::Init { Id: 1, Amount: 100.0f };
closedBill: Bill::Closed = initBill.Close(9001);
```
