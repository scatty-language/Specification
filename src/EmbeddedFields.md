# Embedded Fields

* MAY declare an embedded field within a `Data` type by placing the `embed` modifier immediately after the field's type (`Identifier: Type embed`).
* MUST forward accessible instance members (instance structural fields, instance computed properties, and instance methods, excluding `~Disposable.Dispose`) of the embedded field's type to the enclosing `Data` type.
* MUST apply member forwarding transitively across nested embedded fields.
* MUST NOT forward static methods, static properties, `static`/`astatic`/`gstatic` fields, compile-time constants (`::`), or `~Disposable.Dispose` through an embedded field.
* MUST compute the effective visibility of any member or variant forwarded through an embedded field chain as the most restrictive visibility among the embedded fields along that chain and the target member itself (where `private` < `internal` < `public`).
* MUST NOT forward `private` (`_name`) members of an embedded type.
* MUST NOT forward `internal` (`Name_`) members of an embedded type declared in a different package.
* MUST automatically pass the embedded field by reference when a forwarded method requires a `ref @` receiver.
* MUST evaluate all forwarded member accesses, mutations, and variant switches against the underlying embedded field chain, enforcing all applicable `TypeGuards` and Law of Exclusivity rules on every field along that chain.
* MUST permit direct access to the embedded field via its declared identifier subject to standard visibility rules.
* MUST NOT combine the `embed` modifier with `static`, `astatic`, or `gstatic` on the same field.
* MUST NOT apply the `embed` modifier to an untagged union (`tag none`) or a weak reference type (`T weak`).

## Embedded Variant Fields

* A `Data` type is classified as variant-bearing if it directly declares variants (without `tag none`) or declares an embedded field whose type is variant-bearing and exposes variants accessible to the enclosing `Data` type.
* MUST NOT declare an embedded variant field within a `Data` type that directly declares variants (`::Variant`).
* MUST NOT declare more than one embedded variant field (directly or transitively) within a single `Data` declaration.
* MAY declare an embedded field typed as a specific narrowed variant subtype (`Data::Variant`).
* MUST NOT classify an embedded `Data::Variant` field as a variant-bearing embedded variant field.
* MUST forward the accessible shared members, accessible payload fields of `::Variant`, and valid instance methods of `Data::Variant` (excluding variant-mutating `ref @` methods prohibited on `Data::Variant`) when a `Data` type embeds a `Data::Variant` field.

### Forwarded Variant Subtypes and State Evaluation

* MUST synthesize a forwarded variant subtype `Parent::Variant` for every variant `::Variant` exposed by an accessible embedded variant field chain of `Parent`, with effective visibility equal to the most restrictive visibility among the embedded fields along that chain.
* MUST guarantee that a forwarded variant subtype `Parent::Variant` has the exact same physical memory layout as `Parent`.
* MUST expose the accessible members of `Parent`, the accessible payload fields of `::Variant` (subject to the effective visibility of the embedded field chain), and any accessible constrained-receiver methods (`@::Variant` or `ref @::Variant`) declared on `Parent` or along the embedded variant chain directly on any expression statically typed or narrowed to `Parent::Variant`.
* MUST allow `Parent` and its `with` extension blocks to declare instance methods with variant-constrained receivers (`@::Variant` or `ref @::Variant`) and variant-subtype return types (`-> @::Variant`) corresponding to the variants exposed by its embedded variant field.
* MUST forward state evaluations (`is`, `is not`, `as`, `as?`, `as!`, `assume`, and `match`) applied to `Parent` directly to the active discriminant tag of the embedded variant field, refining or returning `Parent::Variant` (including with the contextual `.Variant` syntax).
* MUST forward active variant switching (`=::`), variant copy-update (`with ::Variant`), `tagof`, and inline `offsetof Parent::Variant.PayloadField` to the embedded variant field chain.
* MUST implicitly downgrade `Parent::Variant` to `Parent` without an explicit cast.
* MUST NOT switch the active variant via `=::`, assign a different variant, reassign or switch any embedded field along the variant chain, or pass `Parent` or any embedded field along the variant chain by `ref` to a method, `fn` closure, or `fnptr` pointer that can mutate the active variant while the expression is statically typed or narrowed to `Parent::Variant`.

### Forwarded Variant Constructors

* MAY construct an instance of `Parent` using a forwarded variant constructor (`Parent::Variant[(args)] [{ struct_init_body }]` or contextual `.Variant[(args)] [{ struct_init_body }]`) if and only if every `Data` type strictly below `Parent` along the embedded variant field chain declares default values for all of its structural fields other than its own embedded variant field (or declares zero such fields), unless the `zeroed` or `noinit` modifier is applied within an `unsafe` block or `unsafe` method.
* MUST reject a forwarded variant constructor at compile time if any `Data` type strictly below `Parent` along the embedded variant field chain contains a structural field (other than its embedded variant field) lacking a default value and neither `zeroed` nor `noinit` is applied.
* MUST treat the embedded variant field of `Parent` as already assigned by the variant clause prior to evaluating `struct_init_body`:
  * Explicit assignment to the embedded variant field within `struct_init_body` MUST be rejected at compile time.
  * Positional field initializers within `struct_init_body` MUST skip the embedded variant field and bind sequentially to the remaining unassigned structural fields of `Parent`.
* MAY omit `struct_init_body` on a forwarded variant constructor if and only if all remaining structural fields of `Parent` declare default values or `Parent` declares no other structural fields.
* MUST infer the enclosing base type `Parent`—not `Parent::Variant`—when binding a forwarded variant constructor via type inference (`:=`).

```scatty
StatusMessage
{
    Message: String embed = "Wow";
    ::Success;
    ::Error(Code: Int);

    GetCode(@::Error) -> Int => @.Code;
}

RequiredStatus
{
    Origin: String;
    ::Ok;
    ::Fail;
}

Response
{
    Status: StatusMessage embed;
    Timestamp: Int;
}

StrictResponse
{
    Status: RequiredStatus embed;
    Timestamp: Int;
}

ErrorReport
{
    Err: StatusMessage::Error embed;

    // Valid: 'StatusMessage::Error' is a fixed variant subtype, not a variant-bearing type
    ::Pending;
    ::Sent;
}

// Valid: all shared fields of 'StatusMessage' have default values
res := Response::Success { Timestamp: 12345 };

// Valid: in-place variant switch preserves 'Timestamp' and 'Status.Message'
res =:: Error(404);

// Valid: narrows 'res' to 'Response::Error' and exposes 'Code' and '@::Error' methods
may res is .Error
{
    Console.WriteLine(res.Code);
    Console.WriteLine(res.GetCode());

    // ERROR: cannot reassign embedded variant field while 'res' is narrowed to 'Response::Error'
    res.Status = .Success;
}

// Valid: transitive forwarding from 'StatusMessage.Message' ('String')
may res.Contains("Wow")
{
    Console.WriteLine("String method forwarded");
}

// Valid: variant and structural pattern matching (including forwarded structural fields)
match res
{
    .Error(404) { Message: "Wow", Timestamp: ts } => Console.WriteLine(ts);
    .Error(code) => Console.WriteLine(code);
    .Success => Console.WriteLine("OK");
}

// ERROR: 'RequiredStatus.Origin' lacks a default value; forwarded constructor is rejected
badStrict := StrictResponse::Ok { Timestamp: 1 };

// Valid: explicit structural initialization when embedded shared fields lack defaults
goodStrict := StrictResponse { Status: .Ok { Origin: "API" }, Timestamp: 1 };

// Valid: embedded 'Data::Variant' alongside parent's own variants
report := ErrorReport::Pending { Err: StatusMessage::Error(500) };
reportCode := report.Code;
```

## Embedded Option Fields

* MUST classify an embedded field of type `T?` (`Option<T>`) as a standard embedded variant field, subject to all Embedded Variant Fields rules.
* MUST NOT forward optional chaining (`?.`, `?[...]`), postfix force-unwrap (`!`), coalescing (`??`), conditional binding (`:=`), or conditional assignment (`=`) to an embedded `Option<T>` field.
* MUST NOT permit the `none` literal to initialize or assign to a `Data` type with an embedded `Option<T>` field.
* MUST NOT implicitly convert a `Data` type with an embedded `Option<T>` field to `T?`, nor implicitly convert `T` or `T?` to the enclosing `Data` type.

```scatty
User
{
    Name: String;
}

Session
{
    CurrentUser: User? embed;
    Id: Int;
}

session := Session::Some(User { Name: "Alice" }) { Id: 1 };
emptySession: Session = .None { Id: 2 };

// Valid: standard variant forwarding ('is', 'match', '=::', and forwarded 'Option<T>' methods)
may session is .Some(user)
{
    Console.WriteLine(user.Name);
}

// Valid: optional operators applied explicitly to the embedded field
maybeName: String? = session.CurrentUser?.Name;

// ERROR: optional operators and conditional binding are not forwarded to 'CurrentUser'
badName := session?.Name;
badUser := session!;
may u := session { }

// ERROR: 'none' cannot initialize 'Session' directly
badSession: Session = none;
```

## Shadowing, Collisions, and Interface Conformance

* MUST override and shadow any accessible forwarded embedded instance method with the identical name when the parent `Data` type declares an instance method of that name.
* MUST override and shadow any accessible forwarded embedded computed property with the identical name when the parent `Data` type declares a computed property of that name.
* MUST reject compilation if the parent `Data` type declares a structural data field with the identical name as any accessible forwarded structural data field or accessible forwarded variant payload field exposed by an embedded field.
* MUST NOT treat inaccessible (`private`, or `internal` from another package) fields of an embedded type as colliding with fields declared on the parent `Data` type.
* MUST reject compilation if the parent `Data` type declares a member whose identifier matches an accessible forwarded member of a different declaration kind (structural field versus computed property, structural field versus method, or computed property versus method, in either direction).
* MUST reject compilation on ambiguous access (including member access, method invocation, and structural pattern matching) if two or more embedded fields within the same `Data` type expose a member with the identical name, regardless of transitive embedding depth.
* MAY resolve ambiguity across multiple embedded fields for a method or computed property by explicitly declaring a shadowing method or computed property of the same kind on the parent `Data` type.
* MUST NOT implicitly cause the parent `Data` type to implement any `~Interface` implemented by its embedded fields.
* MAY satisfy a strong or weak contract method or property of an `~Interface` explicitly declared on the parent `Data` type (`Parent : ~Interface`) using an accessible, unambiguous forwarded instance method or instance property if and only if:
  1. The forwarded member's effective visibility on `Parent` is at least as broad as the contract member's visibility,
  2. The forwarded member's concrete signature matches the contract member's signature after resolving `@` to `Parent`, and
  3. The contract member is not `~Disposable.Dispose`.

```scatty
// PACKAGE: Core
~Printable
{
    Print(@);
}

Buffer
{
    _size: Int = 0;
    Capacity_: Int = 16;
    Name: String = "Default";

    Print(@) => Console.WriteLine(@.Name);
}

// PACKAGE: App
use Core;

Stream : ~Printable
{
    Buf: Buffer embed;

    // Valid: '_size' is private to 'Buffer' and 'Capacity_' is internal to 'Core'
    _size: Int;
    Capacity_: Int;

    // Valid: 'Stream' explicitly declares ': ~Printable', satisfied by forwarded 'Buf.Print'
}

Wrapper<T>
{
    List: [T] embed;
    ListB: [T] embed;

    WowCount: Int;

    // Valid: overrides forwarded method 'Pop'
    Pop(@)
    {
        Console.WriteLine("Intercepted Pop");
    }

    // Valid: overrides forwarded computed property 'Capacity'
    Capacity: Int => @.ListB.Capacity;
}

BadStream
{
    Buf: Buffer embed;

    // ERROR: collides with accessible forwarded structural field 'Buffer.Name'
    Name: String;
}

BadPropertyStream
{
    Buf: Buffer embed;

    // ERROR: cross-kind collision between computed property and forwarded structural field 'Buffer.Name'
    Name: String => "Custom";
}

HiddenStream : ~Printable
{
    _buf: Buffer embed;

    // ERROR: '_buf' is private, so forwarded 'Print' has private effective visibility
    // and cannot satisfy public '~Printable.Print'
}

wrapper := Wrapper<Int> { List: [Int].New(), WowCount: 0, ListB: [Int].New() };

// ERROR: 'Add' is ambiguous across 'List' and 'ListB'
wrapper.Add(1);
```

## Operator Forwarding and Primitive Types

* MUST delegate `==`, `!=`, and `~Hashable` (`Hash(@) -> Ulong`) to an embedded field if the parent `Data` type declares zero `equality` fields and the embedded field's type supports equality and `~Hashable`, in accordance with `Operators`.
* MUST delegate standard arithmetic operators (`+`, `-`, `*`, `/`, `%` and compound assignments `+=`, `-=`, `*=`, `/=`, `%=`) to an embedded field (including an embedded primitive field) if the parent `Data` type declares zero `math` fields and the embedded field's type supports standard arithmetic operators.
* MUST delegate wrapping arithmetic operators (`+&`, `-&`, `*&` and compound assignments `+&=`, `-&=`, `*&=` ) to an embedded field (including an embedded primitive field) if the parent `Data` type declares zero `math` fields and the embedded field's type supports wrapping arithmetic operators.
* MUST require both operands of a delegated binary arithmetic or wrapping arithmetic operator (or compound assignment) to evaluate to the exact same parent `Data` type.
* MUST evaluate a delegated binary arithmetic or wrapping arithmetic operator by applying the operator to the corresponding embedded fields of both operands, copying all remaining non-delegated fields from the left-hand operand, and yielding a new instance of the parent `Data` type.
* MUST evaluate a delegated compound assignment operator (`+=`, `-=`, `*=`, `/=`, `%=`, `+&=`, `-&=`, `*&=` ) by applying the operator in-place to the left-hand operand's embedded field using the right-hand operand's embedded field, leaving all other fields of the left-hand operand unchanged.
* MUST shadow and override embedded field operator delegation if the parent `Data` type explicitly declares any `math` fields (for standard and wrapping arithmetic operators) or `equality` fields (for equality and `~Hashable`).
* MUST NOT implicitly typecast or subtype the parent `Data` type to the type of its embedded field.
* MUST NOT mix the parent `Data` type and the embedded field's type as operands of a delegated operator.

```scatty
Score
{
    Value: Int embed;
    IsRanked: Bool;
}

scoreA := Score { Value: 10, IsRanked: true };
scoreB := Score { Value: 20, IsRanked: false };

// Valid: evaluates 'scoreA.Value + scoreB.Value' and preserves 'scoreA.IsRanked' (true)
// Result is 'Score { Value: 30, IsRanked: true }'
totalScore: Score = scoreA + scoreB;

// Valid: delegates wrapping addition to embedded 'Int' field
wrappedScore: Score = scoreA +& scoreB;

// Invalid: both operands MUST have the exact same type ('Score') — MUST be rejected
badScore := scoreA + 5;

ProcessInt(val: Int) { }

// Invalid: 'Score' does not implicitly convert to 'Int' — MUST be rejected
ProcessInt(scoreA);

// Valid: explicit access to the embedded field
ProcessInt(scoreA.Value);
```
