# Operators

* MUST derive operator implementations on custom `Data` types exclusively via declarative field modifiers (`equality`, `math`) or interface implementations.

## Operator Precedence

* MUST evaluate expressions according to the descending operator precedence hierarchy:

| Level | Category | Operators | Associativity |
| :--- | :--- | :--- | :--- |
| 1 | Primary Access | `.`, `?.`, `!`, `::`, `()`, `[]`, `with` | Left-to-right |
| 2 | Unary & Introspection | `+`, `-`, `bitnot`, `sizeof`, `alignof`, `offsetof`, `nameof`, `tagof` | Right-to-left |
| 3 | Type & Casting | `as`, `as?`, `as!`, `assume`, `is`, `is not`, `^` | Left-to-right |
| 4 | Multiplicative | `*`, `/`, `%` | Left-to-right |
| 5 | Additive | `+`, `-` | Left-to-right |
| 6 | Shift | `bitshl`, `bitshr` | Left-to-right |
| 7 | Bitwise | `bitand`, `bitxor`, `bitor` | Left-to-right |
| 8 | Relational & Equality | `<`, `>`, `<=`, `>=`, `==`, `!=` | Left-to-right |
| 9 | Logical Negation & Control | `not`, `try?`, `try!` | Right-to-left |
| 10 | Logical AND | `and` | Left-to-right |
| 11 | Logical OR | `or` | Left-to-right |
| 12 | Coalescing | `??` | Right-to-left |
| 13 | Conditional | `then ... :` | Right-to-left |
| 14 | Assignment | `=`, `:=`, `+=`, `-=`, `*=`, `/=`, `%=`, `=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`, `=::` | Right-to-left |

* MUST evaluate operators with equal precedence from left to right, except coalescing (`??`), conditional (`then ... :`), and assignment operators which MUST evaluate from right to left.

```scatty
result := a + b as Float == c * d; 
```

## Logical Operators

* MUST require operands of `and`, `or`, and `not` to evaluate to `Bool`.
* MUST evaluate `and`, `or`, and `not` expressions to `Bool`.
* MUST short-circuit evaluation of `and` expressions by omitting evaluation of the right-hand operand when the left-hand operand evaluates to `false`.
* MUST short-circuit evaluation of `or` expressions by omitting evaluation of the right-hand operand when the left-hand operand evaluates to `true`.

```scatty
may not user.IsBanned and (user.Age > 18 or user.HasConsent) 
{

}
```

## Bitwise Operators

* MUST require operands of `bitand`, `bitor`, `bitxor`, `bitnot`, `bitshl`, and `bitshr` to evaluate to scalar integer primitive types.
* MUST perform an arithmetic right shift via `bitshr` when applied to signed integer types, preserving the sign bit.
* MUST perform a logical right shift via `bitshr` when applied to unsigned integer types, shifting zero bits into the most significant positions.

## Identity and Structural Equality

* MUST compare all structural fields of a `Data` type for `==` and `!=` operations if no fields are marked with the `equality` modifier.
* MUST restrict `==` and `!=` operations to fields marked with the `equality` modifier if one or more fields are marked with `equality`.
* MUST ignore all fields not marked with `equality` during `==` and `!=` evaluation when at least one `equality` modifier is present in the `Data` declaration.
* MAY apply the `equality` modifier to variant payload fields.
* MUST compare both the active variant tag and the variant payload when evaluating equality on a variant field marked with `equality`.
* MUST NOT apply the `equality` modifier to a `T weak` field.
* MUST NOT implicitly generate `==` and `!=` operations for a `Data` type containing one or more `T weak` fields unless at least one non-weak field is explicitly marked with the `equality` modifier.
* MUST evaluate `==` and `!=` between two `T weak` expressions by comparing their trivial fields (or `equality`-marked fields) and comparing the ARC Control Block identity of each underlying Owning Type pointer without dereferencing target payloads (`true` if both are live and reference the exact same ARC Control Block allocations with matching trivial fields, or if both are in a disposed/`none` state, and `false` otherwise).
* MUST NOT apply the `equality` modifier to variant payload fields of an untagged union (`tag none`).
* MUST NOT implicitly generate `==` and `!=` operations for an untagged union (`tag none`) unless at least one shared field declared outside the variants is explicitly marked with the `equality` modifier.

```scatty
User 
{
    Id: Int equality;
    Role: String;
    Status: UserStatus equality;
}

userA := User { Id: 1, Role: "Admin", Status: .Active };
userB := User { Id: 1, Role: "Guest", Status: .Active };

isEqual := userA == userB;
```

## Mathematical Operations

* MAY apply the `math` modifier to structural fields of a `Data` type, alone or combined with `equality`.
* MUST NOT apply the `math` modifier to variant payload fields.
* MUST require fields marked with `math` to be of a type that supports native arithmetic operators.
* MUST automatically derive `+`, `-`, `*`, `/`, `%` and their corresponding compound assignments (`+=`, `-=`, `*=`, `/=`, `%=`) for `Data` types declaring at least one field marked with `math`.
* MUST require both operands of derived arithmetic operations to be of the exact same `Data` type.
* MUST evaluate derived arithmetic operations by applying the operator pairwise to corresponding `math` marked fields.
* MUST populate unmarked fields in the resulting instance by copying the values directly from the left-hand operand.
* MUST NOT apply the `math` modifier to a `T weak` field.

```scatty
Transform 
{
    Name: String;
    X: Float math;
    Y: Float math;
}

t1 := Transform { Name: "Start", X: 10.0f, Y: 5.0f };
t2 := Transform { Name: "Offset", X: 5.0f, Y: 5.0f };

result := t1 + t2; 
```

### Arithmetic Overflow and Division by Zero

* MUST require both operands of binary arithmetic operators (`+`, `-`, `*`, `/`, `%`) on primitive numeric types to evaluate to the exact same numeric type.
* MUST reject at compile time any constant integer expression that overflows the target type's representable range or divides/modulos by `0`.
* MUST immediately trigger runtime Abort or Fatal Panic mechanics in both Debug and Release modes when an integer division (`/`) or remainder (`%`) operation is evaluated with a divisor of `0` at runtime.
* MUST evaluate floating-point (`Float` and `Double`) division by `0.0` according to IEEE 754 semantics, yielding `+Infinity`, `-Infinity`, or `NaN` without panicking.
* MUST enforce the following build-mode semantics for runtime integer arithmetic overflow (`+`, `-`, `*`, unary `-`, and signed `MinValue / -1`):
  * In Debug mode, the compiler MUST detect signed and unsigned integer overflow at runtime and immediately trigger a Fatal Panic.
  * In Release mode, the compiler MUST wrap integer overflow using two's complement modular arithmetic by default (with `MinValue % -1` evaluating to `0`).
  * Implementations MUST provide a compiler configuration flag that retains runtime integer overflow checks in Release mode, triggering an immediate process Abort upon overflow.

## Embedded Field Operator Delegation

* MUST delegate `==` and `!=` operators to an embedded field (`Field: Type embed`) if the parent type declares zero `equality` fields and the embedded field type defines equality operators.
* MUST delegate arithmetic operators to an embedded field (`Field: Type embed`) if the parent type declares zero `math` fields and the embedded field type defines arithmetic operators.
* MUST shadow and override embedded field operators if the parent type explicitly declares any `equality` or `math` fields.
* MUST reject compilation with an error if multiple embedded fields within the same type expose conflicting operators and the parent type does not explicitly shadow them.

```scatty
PhysicsData 
{
    Velocity: Float math;
    Hash: Int equality;
}

Player 
{
    Physics: PhysicsData embed;
    Name: String;
}

Enemy 
{
    Physics: PhysicsData embed;
    Id: Int equality;
}

playerA := Player { Physics: PhysicsData { Velocity: 5.0f, Hash: 1 }, Name: "A" };
playerB := Player { Physics: PhysicsData { Velocity: 2.0f, Hash: 1 }, Name: "B" };

isSamePhysics := playerA == playerB;
playerC := playerA + playerB;
```

## Relational Operations

* MUST evaluate relational operators (`<`, `>`, `<=`, `>=`, `==`, `!=`) to `Bool`.
* MUST NOT implicitly generate ordered relational operators (`<`, `>`, `<=`, `>=`) for custom `Data` types without an interface implementation.
* MUST require custom `Data` types to implement the `~Comparable` interface to support ordered relational operators (`<`, `>`, `<=`, `>=`).
* MUST rewrite ordered relational operations on a `~Comparable` type `a` and `b` as follows:
  * `a < b` evaluates as `a.Compare(b) < 0`
  * `a > b` evaluates as `a.Compare(b) > 0`
  * `a <= b` evaluates as `a.Compare(b) <= 0`
  * `a >= b` evaluates as `a.Compare(b) >= 0`
* MUST require custom `Data` types to implement the `~Comparable` interface to perform ordered comparisons.

```scatty
~Comparable 
{
    Compare(@, other: @) -> Int;
}
```

## Cascade Operator

* MAY use the cascade operator (`..`) at the statement level to invoke multiple methods sequentially on the same receiver instance (`target.MethodA()..MethodB()`).
* MUST evaluate the receiver expression preceding the initial `.` access once and apply all subsequent `..` invocations to that same receiver.
* MUST NOT use the cascade operator inside nested expressions or argument lists.

## Indexing Operator

* MUST require a type to implement the standard library `~Indexable<TKey, TValue>` interface to support the index operator (`[]`).

```scatty
~Indexable<TKey, TValue> 
{
    Get(@, key: TKey) throws -> TValue;
    Set?(ref @, key: TKey, value: TValue) throws;

    At(@, key: TKey) -> TValue? 
    {
        may res := try? @.Get(key) => return .Some(res);
        return .None;
    }

    ^Mutable = @ must @ is .Set;
}
```

### Reading by Index

* MAY omit the `throws` modifier on a concrete `Get(@, key: TKey) -> TValue` implementation to declare an infallible index read.
* MUST invoke `Get` directly without requiring a `throws` context when applying `col[key]` to a statically known infallible implementation.
* MUST invoke `Get` and propagate any thrown exception as a `throws` expression (or immediately trigger Abort/Panic in a non-throwing method) when applying `col[key]` to a fallible implementation.
* MAY invoke `.At(key)` to safely query a fallible indexable collection and receive `TValue?` (`Option::Some` if valid, or `Option::None` if out of bounds or unmapped) without throwing an exception.

### Writing and In-Place Mutation by Index

* MUST require the target type to implement the weak contract `Set(ref @, key: TKey, value: TValue)` (or be constrained by `~Indexable<TKey, TValue>^Mutable`) to support index assignment (`col[key] = value`) and indexed mutations.
* MUST reject index assignment (`col[key] = value`) and indexed mutations at compile time if the concrete type omits `Set`.
* MUST execute compound assignments (`col[key] += val`), field mutations (`col[key].Field = val`), variant switches (`col[key] =:: Variant`), mutating method calls (`col[key].Mutate()`), and `ref` argument bindings (`Func(ref col[key])`) on built-in addressable collections (`[T]`, `[T; N]`, `Array<T>`, `[K: V]`, `[K -> V]`, and field projections of `soa` collections `soaCol[i].Field`) directly in-place against the element's memory slot:
    1. Detach the collection's underlying Owning Type buffer via Copy-on-Write if its `strongCount > 1`.
    2. Evaluate `key` once and resolve the target element's memory slot in a single lookup.
    3. Apply the mutation or bind the `ref` parameter directly to the resolved memory slot without invoking `Get` or `Set` and without incrementing `strongCount` of the element or its nested Owning Type fields.
* MUST automatically unpack compound assignments (`col[key] += val`), field mutations (`col[key].Field = val`), variant switches (`col[key] =:: Variant`), mutating method calls (`col[key].Mutate()`), and `ref` argument bindings (`Func(ref col[key])`) on custom `Data` types implementing `~Indexable<TKey, TValue>` with `Set` into a single-evaluation read-modify-write sequence:
  1. Evaluate `key` once into a temporary binding `_key`.
  2. Read `_temp := col.Get(_key)`.
  3. Apply the mutation or `ref` argument invocation to `_temp`.
  4. Write the modified value back via `col.Set(_key, _temp)`.

### Multi-Argument Indexing

* MAY supply two or more comma-separated index expressions within brackets (`col[k1, k2, ...]`).
* MUST implicitly pack multiple index arguments `(k1, k2, ...)` into a tuple of type `(T1, T2, ...)` and resolve the operation against `~Indexable<(T1, T2, ...), TValue>`.

```scatty
UpdateHero(ref heroes: [String: Hero], ref grid: Grid) 
{
    // Safe optional lookup via '.At()' combined with optional chaining '?.' and '??'
    hp: Int = heroes.At("Arthas")?.Hp ?? 0;

    // Built-in collection: single lookup, direct in-place slot mutation (strongCount remains 1)
    heroes["Arthas"].Hp -= 10;
    heroes["Arthas"].Inventory.Add("Frostmourne");

    // Custom '~Indexable' type: unpacked into Get + mutate '_temp' + Set
    grid[2, 3].Value += 1;

    // Multi-argument indexing packs '(2, 3)' into a tuple key
    grid[2, 3] = Cell { Value: 99 };
    cell := grid[2, 3];
}
```

## Conditional Operator

* MAY evaluate a conditional expression using the ternary syntax `condition then true_expr : false_expr`.
* MUST require `condition` to evaluate to `Bool`.
* MUST short-circuit evaluation by evaluating and returning `true_expr` (omitting `false_expr`) when `condition` evaluates to `true`.
* MUST short-circuit evaluation by evaluating and returning `false_expr` (omitting `true_expr`) when `condition` evaluates to `false`.
* MUST require `true_expr` and `false_expr` to evaluate to the exact same type `T`, except when one branch evaluates to `T` and the other evaluates to `T?` or `none`, in which case the expression MUST evaluate to `T?` (`Option<T>`) and implicitly wrap the `T` branch into `Option::Some`.
* MUST associate chained conditional operators from right to left (`c1 then v1 : c2 then v2 : v3` evaluates as `c1 then v1 : (c2 then v2 : v3)`).

```scatty
label := score >= 50 then "Pass" : "Fail";

// Infers 'Char?' by combining 'Char' and 'none'
cell := (x + y) % 11 == 0 then '.' : none;

// Right-associative chaining
tier := points >= 100 then "Gold" : points >= 50 then "Silver" : "Bronze";
```

## Optional Chaining, Force Unwrap, and Coalescing

### Optional Chaining

* MAY apply the optional chaining operators (`?.` and `?[...]`) to any expression of an Option type (`T?`) or a weak reference type (`T weak`).
* MUST reject applying `?.` or `?[...]` to a non-optional, non-weak type at compile time.
* MUST evaluate the left-hand target expression once:
    * If the target evaluates to `Option::Some(val)` (or a live `T weak` where all underlying `strongCount > 0`), the compiler MUST unwrap (or temporarily promote) `val` and evaluate the remainder of the postfix chain.
    * If the target evaluates to `Option::None` (or a disposed `T weak` where any underlying `strongCount == 0`), the compiler MUST short-circuit and skip the entire remaining postfix chain (`.A.B.C()`), immediately yielding `Option::None` (or performing a no-op if the final member returns void).
* MUST wrap a non-optional return value `R` of an optionally chained expression into `R?` (`Option<R>`).
* MUST flatten the result of an optionally chained expression to `R?` (rather than `R??`) if the invoked member already returns an optional type `R?`.
* MAY invoke a mutating (`ref @`) method via `?.` on a mutable `T?` lvalue variable, mutating the underlying `Some` payload in-place if present.

```scatty
User 
{
    Name: String;
    Score: Int;

    AddBonus(ref @, bonus: Int) => @.Score += bonus;
    GetNickname(@) -> String? => .Some(@.Name);
}

maybeUser: User? = GetUser();

// Evaluates to 'Int?' ('Option::Some' if present, 'Option::None' if absent)
score: Int? = maybeUser?.Score;

// Flattens 'String?' return type (yields 'String?', NOT 'String??')
nick: String? = maybeUser?.GetNickname();

// Mutates 'maybeUser' payload in-place if 'Some'; no-op if 'None'
maybeUser?.AddBonus(5);
```

### Postfix Force Unwrap

* MAY apply the postfix force-unwrap operator (`!`) to any expression of an Option type (`T?`) or a weak reference type (`T weak`).
* MUST evaluate `expr!` to the unwrapped payload `T` if `expr` is `Option::Some` (or a live `T weak` where all underlying `strongCount > 0`).
* MUST immediately trigger runtime Abort or Fatal Panic mechanics if `expr` evaluates to `Option::None` (or a disposed `T weak` where any underlying `strongCount == 0`).
* MAY chain member accesses or invocations directly after `!` (`expr!.Method()`).

```scatty
optInt: Int? = .Some(10);

// Unwraps directly to 'Int' (or Aborts/Panics if 'None')
exactVal: Int = optInt!;

// Force-unwraps and invokes a method on the underlying 'Int'
result: Int = optInt!.Max(5);
```

### Coalescing Operator

* MAY apply the binary coalescing operator (`left ?? right`) where `left` is an Option type (`T?`) or a weak reference type (`T weak`), and `right` evaluates to `T` (or `T?` for chained coalescing).
* MUST evaluate `left` first and short-circuit (omitting evaluation of `right`) if `left` yields `Option::Some(val)` (or a live `T weak`), returning `val` of type `T`.
* MUST evaluate and return `right` if and only if `left` yields `Option::None` (or a disposed `T weak` where any underlying `strongCount == 0`).

## Introspection Operators

### `sizeof` and `alignof`

* MUST apply `sizeof` and `alignof` exclusively to a `Type` or to the self-type token `@` (`sizeof Type`, `sizeof @`, `alignof Type`, `alignof @`).
* MUST NOT apply `sizeof` or `alignof` to value expressions or variable identifiers.
* MUST restrict `sizeof @` and `alignof @` to the lexical scope of a `Data` declaration, `~Interface` declaration, `with` extension block, or `^Guard` declaration.
* MUST evaluate `sizeof` at compile time to the total byte footprint of the target type, including all internal and trailing alignment padding.
* MUST evaluate `alignof` at compile time to the required byte alignment boundary of the target type.
* MUST infer the result of `sizeof` and `alignof` as `Int` when the target type is inferred (`:=`) without an expected integer type context.
* MUST contextually type the result of `sizeof` and `alignof` to the expected scalar integer primitive type (`Int`, `Uint`, `Long`, `Ulong`, `Size`, `Usize`, or `Byte`) when the target type is statically known from context, rejecting the expression at compile time if the computed value exceeds the representable range of the expected type.
* MUST evaluate `sizeof` and `alignof` on an Owning Type (`~Disposable`) to the byte size and byte alignment of its inline pointer representation (`sizeof Usize` and `alignof Usize`).
* MUST evaluate `sizeof` and `alignof` on a variant subtype (`Data::Variant`) or a guarded type (`Type^Guard`) to the exact byte size and byte alignment of the underlying base `Data` type.
* MUST resolve `sizeof` and `alignof` applied to a generic type parameter (`sizeof T`, `alignof T`) at compile time during monomorphization for each concrete type argument.
* MUST reject applying `sizeof` or `alignof` directly to a bare `~Interface` type at compile time.

```scatty
Vector(X: Int, Y: Int);

// Inferred as 'Int' by default (evaluates to 8 and 4)
vecSize := sizeof Vector;
vecAlign := alignof Vector;

// Contextually typed as 'Usize'
rawBytes: Usize = sizeof Vector;

// Data wrapping a single 'Array<Char>' Owning Type field: evaluates to pointer size (8 on 64-bit)
strSlotSize := sizeof String;

// ERROR: Cannot apply 'sizeof' to a variable or expression
v := Vector { X: 1, Y: 2 };
badSize := sizeof v;

// ERROR: Bare interface has no concrete memory layout
badInterfaceSize := sizeof ~Printable;
```

### `offsetof`

* MUST apply `offsetof` to a static field path rooted in a concrete `Data` type, variant subtype (`Data::Variant`), or `@` (`offsetof Type.Field`, `offsetof Data::Variant.PayloadField`, `offsetof @.Field`).
* MUST permit chained field paths (`offsetof Type.FieldA.FieldB`) if and only if every intermediate field is an inline non-`~Disposable` `Data` type.
* MUST evaluate `offsetof` at compile time to the byte offset of the target field from byte `0` of the root `Data` instance.
* MUST infer the result of `offsetof` as `Int` by default and contextually type it to any expected scalar integer primitive type following the same rules as `sizeof` and `alignof`.
* MUST NOT apply `offsetof` to:
  * Bitfields (`bitwidth N`),
  * Anonymous padding fields (`_`),
  * Computed properties, methods, or compile-time constants (`::`),
  * `static`, `astatic`, or `gstatic` fields,
  * Fields accessed through an intermediate Owning Type (`~Disposable`), `weak` reference, or `soa` collection, or
  * Bare `~Interface` types.

```scatty
HttpResponse 
{
    Timestamp: Long;
    ::Success;
    ::Error(Code: Int, Message: String);

    CodeOffset() -> Int => offsetof @::Error.Code;
}

tsOffset := offsetof HttpResponse.Timestamp;       // Evaluates to 0
errCodeOffset := offsetof HttpResponse::Error.Code; // Byte offset of 'Code' inside payload union
```

### `tagof`

* MAY apply `tagof` to an expression of a tagged `Data` type or variant subtype (`tagof instance`, `tagof @`), or to a variant type path (`tagof Data::Variant`).
* MUST evaluate `tagof` to the declared discriminant tag type of the target `Data` type (`Int` by default, or the explicit scalar integer primitive `T` specified via `tag T`).
* MUST evaluate `tagof Data::Variant` (and `tagof expr` when `expr` is a compile-time constant or statically narrowed to `Data::Variant`) at compile time as a constant expression yielding that variant's discriminant tag value.
* MUST read and return the active discriminant tag value at runtime when `tagof` is applied to an unnarrowed runtime instance of a tagged `Data` type.
* MUST evaluate `tagof` on a pointer-niche-optimized `Option<T>` (`T?`) instance to `0` when the value is `Option::None` and `1` when the value is `Option::Some`.
* MUST reject applying `tagof` to an untagged union (`tag none`), a `Data` type that declares no variants, a primitive type, a tuple, or an `~Interface` at compile time.

```scatty
PacketType tag Byte 
{
    ::Ping = 1;
    ::Data(Payload: [Byte]) = 2;
    ::Ack  = 5;
}

// Compile-time constant evaluation on a variant path (value 5)
AckCode: Byte :: tagof PacketType::Ack;

InspectPacket(pkt: PacketType) 
{
    // Runtime evaluation of the active discriminant tag (typed as 'Byte')
    rawTag: Byte = tagof pkt;
    Console.WriteLine("Packet tag: \(rawTag)");
}
```

### `nameof`

* MAY apply `nameof` to a type, interface, type guard, variant, local variable, parameter, constant, or member access chain (`nameof Target`).
* MUST evaluate `nameof` at compile time to a constant `String` literal containing the unqualified lexical identifier of the final segment in `Target`.
* MUST strip leading prefix sigils (`~` on interfaces, `^` on type guards, `::` on variants, and enclosing backticks `` ` `` on escaped keywords) from the resulting `String`.
* MUST include formatted generic type arguments (`<...>`) in the resulting `String` when `Target` terminates with a generic type or generic method specialization (`nameof Cell<Int>` evaluates to `"Cell<Int>"`).
* MUST substitute concrete monomorphized type arguments into the resulting `String` when `nameof` references an active generic parameter (`T`) or `@` within a generic type or method.
* MUST validate that the target identifier or member chain resolves to a valid declaration in the current scope at compile time without evaluating any runtime expressions or triggering side effects.

```scatty
Cell<T> 
{
    Value: T;

    TypeName(@) -> String => nameof @;
}

^Positive = Int must @ > 0;

~Printable 
{
    Print(@);
}

Color 
{
    ::Red;
    ::Green;
}

cell := Cell<Int> { Value: 42 };
`match` := 10;

a: String :: nameof Cell<Int>;       // "Cell<Int>"
b: String :: nameof cell.Value;      // "Value"
c: String :: nameof ~Printable;      // "Printable"
d: String :: nameof ^Positive;       // "Positive"
e: String :: nameof Int^Positive;    // "Positive"
f: String :: nameof Color::Red;      // "Red"
g: String :: nameof `match`;         // "match"
h := cell.TypeName();                // "Cell<Int>"
```
