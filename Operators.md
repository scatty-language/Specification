# Operators

* MUST derive operator implementations on custom `Data` types exclusively via declarative field markers (`$`, `#`) or interface implementations.

## Operator Precedence

* MUST evaluate expressions according to the descending operator precedence hierarchy:

| Level | Category | Operators | Associativity |
| :--- | :--- | :--- | :--- |
| 1 | Primary Access | `.`, `?.`, `!`, `::`, `()`, `[]`, `with` | Left-to-right |
| 2 | Unary Arithmetic & Bitwise | `+`, `-`, `bnot` | Right-to-left |
| 3 | Type & Casting | `as`, `as?`, `as!`, `unas`, `is`, `is not`, `^` | Left-to-right |
| 4 | Multiplicative | `*`, `/`, `%` | Left-to-right |
| 5 | Additive | `+`, `-` | Left-to-right |
| 6 | Shift | `bsl`, `bsr` | Left-to-right |
| 7 | Bitwise | `band`, `bxor`, `bor` | Left-to-right |
| 8 | Relational & Equality | `<`, `>`, `<=`, `>=`, `==`, `!=` | Left-to-right |
| 9 | Logical Negation & Control | `not`, `try?`, `try!` | Right-to-left |
| 10 | Logical AND | `and` | Left-to-right |
| 11 | Logical OR | `or` | Left-to-right |
| 12 | Coalescing | `??` | Right-to-left |
| 13 | Assignment | `=`, `:=`, `+=`, `-=`, `*=`, `/=`, `%=`, `=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^`, `=::` | Right-to-left |

* MUST evaluate binary operators with equal precedence from left to right, except coalescing (`??`) and assignment operators which MUST evaluate from right to left.

```C#
result := a + b as Float == c * d; 
```

## Logical Operators

* MUST require operands of `and`, `or`, and `not` to evaluate to `Bool`.
* MUST evaluate `and`, `or`, and `not` expressions to `Bool`.
* MUST short-circuit evaluation of `and` expressions by omitting evaluation of the right-hand operand when the left-hand operand evaluates to `false`.
* MUST short-circuit evaluation of `or` expressions by omitting evaluation of the right-hand operand when the left-hand operand evaluates to `true`.

```C#
if not user.IsBanned and (user.Age > 18 or user.HasConsent) 
{

}
```

## Bitwise Operators

* MUST require operands of `band`, `bor`, `bxor`, `bnot`, `bsl`, and `bsr` to evaluate to scalar integer primitive types.
* MUST perform an arithmetic right shift via `bsr` when applied to signed integer types, preserving the sign bit.
* MUST perform a logical right shift via `bsr` when applied to unsigned integer types, shifting zero bits into the most significant positions.

## Identity and Structural Equality

* MUST compare all structural fields of a `Data` type for `==` and `!=` operations if no fields are marked with the `$` modifier.
* MUST restrict `==` and `!=` operations to fields marked with the `$` modifier if one or more fields are marked with `$`.
* MUST ignore all fields not marked with `$` during `==` and `!=` evaluation when at least one `$` modifier is present in the `Data` declaration.
* MAY apply the `$` modifier to variant fields.
* MUST compare both the active variant tag and the variant payload when evaluating equality on a variant field marked with `$`.
* MUST NOT apply the `$` modifier to a `T weak` field.
* MUST NOT implicitly generate `==` and `!=` operations for a `Data` type containing one or more `T weak` fields unless at least one non-weak field is explicitly marked with the `$` modifier.
* MUST evaluate `==` and `!=` between two `T weak` expressions by comparing their ARC Control Block identity without dereferencing the target payload (`true` if both reference the exact same live allocation with `strongCount > 0` or if both have `strongCount == 0`, and `false` otherwise).
* MUST NOT apply the `$` modifier to variant payload fields of an untagged union (`::: none`).
* MUST NOT implicitly generate `==` and `!=` operations for an untagged union (`::: none`) unless at least one shared field declared outside the variants is explicitly marked with the `$` modifier.

```C#
User 
{
    $Id: Int;
    Role: String;
    $Status: UserStatus;
}

userA := User { Id: 1, Role: "Admin", Status: .Active };
userB := User { Id: 1, Role: "Guest", Status: .Active };

isEqual := userA == userB;
```

## Mathematical Operations

* MAY apply the `#` modifier to structural fields of a `Data` type.
* MUST NOT apply the `#` modifier to variant fields.
* MUST require fields marked with `#` to be of a type that supports native arithmetic operators.
* MUST automatically derive `+`, `-`, `*`, `/`, `%` and their corresponding compound assignments (`+=`, `-=`, `*=`, `/=`, `%=`) for `Data` types declaring at least one field marked with `#`.
* MUST require both operands of derived arithmetic operations to be of the exact same `Data` type.
* MUST evaluate derived arithmetic operations by applying the operator pairwise to corresponding `#` marked fields.
* MUST populate unmarked fields in the resulting instance by copying the values directly from the left-hand operand.
* MUST NOT apply the `#` modifier to a `T weak` field.

```C#
Transform 
{
    Name: String;
    #X: Float;
    #Y: Float;
}

t1 := Transform { Name: "Start", X: 10.0f, Y: 5.0f };
t2 := Transform { Name: "Offset", X: 5.0f, Y: 5.0f };

result := t1 + t2; 
```

## Embedded Field Operator Delegation

* MUST delegate `==` and `!=` operators to an embedded field (`Field embed`) if the parent type declares zero `$` fields and the embedded field type defines equality operators.
* MUST delegate arithmetic operators to an embedded field (`Field embed`) if the parent type declares zero `#` fields and the embedded field type defines arithmetic operators.
* MUST shadow and override embedded field operators if the parent type explicitly declares any `$` or `#` fields.
* MUST reject compilation with an error if multiple embedded fields within the same type expose conflicting operators and the parent type does not explicitly shadow them.

```C#
PhysicsData 
{
    #Velocity: Float;
    $Hash: Int;
}

Player 
{
    Physics: PhysicsData embed;
    Name: String;
}

Enemy 
{
    Physics: PhysicsData embed;
    $Id: Int;
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

```C#
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

```C#
~Indexable<TKey, TValue> 
{
    Get(@, key: TKey) throws -> TValue;
    Set?(ref @, key: TKey, value: TValue) throws;

    At(@, key: TKey) -> TValue? 
    {
        if res := try? @.Get(key) => return .Some(res);
        return .None;
    }

    ^Mutable = @ where @ is .Set;
}
```

### Reading by Index

* MAY omit the `throws` modifier on a concrete `Get(@, key: TKey) -> TValue` implementation to declare an infallible index read.
* MUST invoke `Get` directly without requiring a `throws` context when applying `col[key]` to a statically known infallible implementation.
* MUST invoke `Get` and propagate any thrown exception as a `throws` expression (or immediately trigger Abort/Panic in a non-throwing method) when applying `col[key]` to a fallible implementation.
* MAY invoke `.At(key)` to safely query a fallible indexable collection and receive `TValue?` (`Option::Some` if valid, or `Option::None` if out of bounds or unmapped) without throwing an exception.

### Writing and In-Place Mutation by Index

* MUST require the target type to implement the weak contract `Set(ref @, key: TKey, value: TValue)` (or be constrained by `~Indexable<TKey, TValue>^Mutable`) to support index assignment (`col[key] = value`).
* MUST reject index assignment (`col[key] = value`) at compile time if the concrete type omits `Set`.
* MUST automatically unpack compound assignments (`col[key] += val`), field mutations (`col[key].Field = val`), or mutating method calls (`col[key].Mutate()`) on non-contiguous `~Indexable` types implementing `Set` into a single-evaluation read-modify-write sequence:
  1. Evaluate `key` once into a temporary binding `_key`.
  2. Read `_temp := col.Get(_key)`.
  3. Apply the field mutation, compound assignment, or `ref @` method call to `_temp`.
  4. Write the modified value back via `col.Set(_key, _temp)`.

### Multi-Argument Indexing

* MAY supply two or more comma-separated index expressions within brackets (`col[k1, k2, ...]`).
* MUST implicitly pack multiple index arguments `(k1, k2, ...)` into a tuple of type `(T1, T2, ...)` and resolve the operation against `~Indexable<(T1, T2, ...), TValue>`.

```C#
UpdateHero(ref heroes: [String $ Hero], ref grid: Grid) 
{
    // Safe optional lookup via '.At()' combined with optional chaining '?.' and '??'
    hp: Int = heroes.At("Arthas")?.Hp ?? 0;

    // Automatic read-modify-write unpacking via Get + Set:
    heroes["Arthas"].Hp -= 10;

    // Multi-argument indexing packs '(2, 3)' into a tuple key
    grid[2, 3] = 99;
    cell := grid[2, 3];
}
```

## Optional Chaining, Force Unwrap, and Coalescing

### Optional Chaining

* MAY apply the optional chaining operators (`?.` and `?[...]`) to any expression of an Option type (`T?`) or a weak reference type (`T weak`).
* MUST reject applying `?.` or `?[...]` to a non-optional, non-weak type at compile time.
* MUST evaluate the left-hand target expression once:
  * If the target evaluates to `Option::Some(val)` (or a live `T weak` with `strongCount > 0`), the compiler MUST unwrap (or temporarily retain) `val` and evaluate the remainder of the postfix chain.
  * If the target evaluates to `Option::None` (or a disposed `T weak` with `strongCount == 0`), the compiler MUST short-circuit and skip the entire remaining postfix chain (`.A.B.C()`), immediately yielding `Option::None` (or performing a no-op if the final member returns void).
* MUST wrap a non-optional return value `R` of an optionally chained expression into `R?` (`Option<R>`).
* MUST flatten the result of an optionally chained expression to `R?` (rather than `R??`) if the invoked member already returns an optional type `R?`.
* MAY invoke a mutating (`ref @`) method via `?.` on a mutable `T?` lvalue variable, mutating the underlying `Some` payload in-place if present.

```C#
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
* MUST evaluate `expr!` to the unwrapped payload `T` if `expr` is `Option::Some` (or a live `T weak` with `strongCount > 0`).
* MUST immediately trigger runtime Abort or Fatal Panic mechanics if `expr` evaluates to `Option::None` (or a disposed `T weak` with `strongCount == 0`).
* MAY chain member accesses or invocations directly after `!` (`expr!.Method()`).

```C#
optInt: Int? = .Some(10);

// Unwraps directly to 'Int' (or Aborts/Panics if 'None')
exactVal: Int = optInt!;

// Force-unwraps and invokes a method on the underlying 'Int'
result: Int = optInt!.Max(5);
```

### Coalescing Operator

* MAY apply the binary coalescing operator (`left ?? right`) where `left` is an Option type (`T?`) or a weak reference type (`T weak`), and `right` evaluates to `T` (or `T?` for chained coalescing).
* MUST evaluate `left` first and short-circuit (omitting evaluation of `right`) if `left` yields `Option::Some(val)` (or a live `T weak`), returning `val` of type `T`.
* MUST evaluate and return `right` if and only if `left` yields `Option::None` (or a disposed `T weak` with `strongCount == 0`).
