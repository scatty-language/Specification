# Type Transformations

* MUST implement the standard library `~Castable<T>` interface on a user-defined `Data` type to support custom type transformations via the `as`, `as?`, and `as!` operators.
* MUST NOT apply the `unas` operator to any `~Castable<T>` transformation.

```C#
~Castable<T> 
{
    Cast(@) throws -> T;
}
```

## Infallible Transformations

* MAY omit the `throws` modifier on the concrete `Cast(@) -> T` method implementation to declare an infallible transformation.
* MUST invoke the `Cast` method directly without requiring a `throws` context or `try` construct when applying `as` to a statically known infallible implementation.
* MUST NOT apply `as?` or `as!` to a statically known concrete type whose `Cast(@) -> T` implementation omits `throws`.
* MAY apply `as`, `as?`, and `as!` to a generic type parameter constrained by `~Castable<T>`.
* MUST evaluate `as?` to `Option::Some` and `as!` to the transformed value without panicking when a generic parameter constrained by `~Castable<T>` is monomorphized with an infallible type.

## Fallible Transformations

* MUST declare the `throws` modifier on the concrete `Cast(@) throws -> T` method implementation to declare a fallible transformation.
* MUST invoke `Cast` and propagate any thrown exception as a `throws` expression when applying the `as` operator.
* MUST invoke `Cast`, intercept any thrown exception, and evaluate to `T?` (`Option::Some` on success, `Option::None` on failure) when applying the `as?` operator.
* MUST invoke `Cast` and immediately trigger runtime Abort or Fatal Panic mechanics if an exception is thrown when applying the `as!` operator.

## Primitive Types Transformation

* MUST provide intrinsic implementations of infallible `~Castable<T>` for safe widening primitive conversions (e.g., `Int` to `Float`, `Int` to `Long`).
* MUST provide intrinsic implementations of fallible `~Castable<T>` for narrowing primitive conversions (e.g., `Long` to `Int`).

## Chained Type Guard Transformations

* MAY apply `as`, `as?`, or `as!` to transform a source type directly into a guarded target type (`Source as Target^Guard`) if the source type implements `~Castable<Target>`.
* MUST execute the `~Castable<Target>` transformation prior to evaluating the `^Guard` invariant.

```C#
Vector : ~Castable<Point> 
{
    X: Int;
    Y: Int;

    // Infallible implementation (omits 'throws')
    Cast(@) -> Point 
    {
        return Point { X: @.X, Y: @.Y };
    }
}

vec := Vector { X: 10, Y: 20 };

// Allocates a new 'Point' instance
point := vec as Point; 

// ERROR: Statically known infallible conversions do not support 'as?' or 'as!'
bad := vec as? Point; 

Record : ~Castable<User> 
{
    RawId: String;

    // Fallible implementation (retains 'throws')
    Cast(@) throws -> User 
    {
        may @.RawId == "" => throw Exception.New("Empty ID");
        
        parsedId := @.RawId as! Int; 
        
        return User { Id: parsedId };
    }
}

record := Record { RawId: "123" };

// Fallible 'as' acts as a throws expression
ProcessRecord() throws 
{
    user := record as User; 
}

// Safe inline unpacking using 'as?'
may user := record as? User 
{
    Console.WriteLine(user.Id);
}

// Forced transformation using 'as!'
validUser := record as! User; // Aborts/Panics immediately if 'RawId' is invalid

// Chained transformation and type guard validation
validId := "42" as? Int^Positive;
```
