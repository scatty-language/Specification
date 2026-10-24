# Operators

* MUST derive operator implementations on custom `Data` types exclusively via declarative field markers (`$`, `#`) or interface implementations.

## Operator Precedence

* MUST evaluate expressions according to the descending operator precedence hierarchy:

| Level | Category | Operators | Associativity |
| :--- | :--- | :--- | :--- |
| **1** | Primary Access | `.`, `::`, `()`, `[]` | Left-to-right |
| **2** | Unary & Control | `not`, `bnot`, `try?`, `try!` | Right-to-left |
| **3** | Type & Casting | `as`, `as?`, `as!`, `unas`, `is`, `^` | Left-to-right |
| **4** | Multiplicative | `*`, `/`, `%` | Left-to-right |
| **5** | Additive | `+`, `-` | Left-to-right |
| **6** | Shift | `bsl`, `bsr` | Left-to-right |
| **7** | Bitwise | `band`, `bxor`, `bor` | Left-to-right |
| **8** | Relational & Equality | `<`, `>`, `<=`, `>=`, `==`, `!=` | Left-to-right |
| **9** | Logical AND | `and` | Left-to-right |
| **10** | Logical OR | `or` | Left-to-right |
| **11** | Assignment | `=`, `:=`, `+=`, `-=`, `*=`, `/=`, `%=`, `=^`, `+=^`, `-=^`, `*=^`, `/=^`, `%=^` | Right-to-left |

* MUST evaluate binary operators with equal precedence from left to right, except assignment operators which MUST evaluate from right to left.

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

* MUST delegate `==` and `!=` operators to an embedded field (`@Field`) if the parent type declares zero `$` fields and the embedded field type defines equality operators.
* MUST delegate arithmetic operators to an embedded field (`@Field`) if the parent type declares zero `#` fields and the embedded field type defines arithmetic operators.
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

## Cascade Operator (`..`)

* MAY use the cascade operator (`..`) at the statement level to invoke multiple methods sequentially on the same receiver instance (`target.MethodA()..MethodB()`).
* MUST evaluate the receiver expression preceding the initial `.` access once and apply all subsequent `..` invocations to that same receiver.
* MUST NOT use the cascade operator inside nested expressions or argument lists.
