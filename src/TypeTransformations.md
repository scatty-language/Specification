# Type Transformations

* MUST implement the standard library `~Castable<T>` interface on a user-defined `Data` type to support custom type transformations via the `as`, `as?`, and `as!` operators.
* MAY implement multiple specializations of `~Castable<T>` on the same `Data` type, declaring a corresponding `Cast(@) -> T` or `Cast(@) throws -> T` method for each target type `T`.
* MUST NOT specify a guarded type (`Target^Guard`) as the type argument `T` when implementing `~Castable<T>`.
* MUST NOT apply the `assume` operator to any `~Castable<T>` transformation.

```scatty
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
* MUST execute the `~Castable<Target>` transformation to obtain the unguarded `Target` value prior to evaluating the `^Guard` invariant.

```scatty
^Positive = Int must @ > 0;

// ERROR: Cannot implement '~Castable<T>' for a guarded target type 'Int^Positive'
BadCaster : ~Castable<Int^Positive> 
{
    Cast(@) -> Int^Positive => 1 as ^;
}

Metric : ~Castable<Int>, ~Castable<Float> 
{
    Raw: String;

    // Fallible conversion to 'Int'
    Cast(@) throws -> Int => @.Raw as Int;

    // Infallible conversion to 'Float'
    Cast(@) -> Float => 0.0f;
}

metric := Metric { Raw: "42" };

// Resolves 'Cast(@) -> Float' (infallible)
f := metric as Float;

// Resolves 'Cast(@) throws -> Int' followed by '^Positive' guard validation
validId := metric as? Int^Positive;
```
