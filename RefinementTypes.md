# Refinement Types

```EBNF
RefinementDecl ::= "^" Identifier ["<" TypeParams ">"] "=" TargetType [":" ParentList] "where" Expression ";"
TargetType     ::= "@" | Type
TypeParams     ::= TypeParameter ("," TypeParameter)*
RefinedType ::= BaseType "^" Identifier ["<" TypeArgs ">"]
```

* MUST declare refinement types at the file's top level.
* MUST evaluate the `where` clause expression to a pure, side-effect-free boolean.
* MUST explicitly specify the base type being refined immediately following the `=` token.
* MAY specify `@` as the `TargetType` to create a universal refinement applicable to any type satisfying the `where` expression.
* MUST evaluate `@` within the `where` expression as the instance receiver.
* MAY declare explicit generic parameters and assign a type parameter as the `TargetType`.

```C#
^Positive = @ where @ > 0;

^PositiveNumber<T: ~INumber> = T where @ > 0;

^Email = String where @.Contains("@");
```

## Refinement Inheritance and Composition

* MAY inherit from other refinements using the `:` operator.
* MUST implicitly forward the current refinement's base type to inherited parent refinements.
* MAY provide explicit generic arguments to parent refinements using `<@>` or concrete type identifiers.

```C#
^Positive = @ where @ > 0;

^Big = @ : ^Positive where @ > 1000;

^Complex = @ : ^Positive, ^AnotherComplex<@> where @ % 2 == 0;
```

## Applying Refinements

* MUST apply a refinement to a base type using the postfix `^` operator.
* MUST define the memory layout solely via the `BaseType`.
* MUST evaluate the `^Refinement` strictly as a compile-time or runtime invariant.
* MUST implicitly pass the `BaseType` as the primary target type argument to the applied refinement.

```C#
validAge: Int^Positive = 25;
email: String^Email = "test@domain.com";

ProcessAge(age: Int^Positive) { }

ProcessBigNumber(num: Float^Big) { }
```

## Validation and Casting

* MAY implicitly cast a refined type to its base type.
* MUST NOT implicitly cast a base type to a refined type.
* MUST explicitly validate the invariant when casting from a base type to a refined type.

### Operators

* MUST evaluate the invariant and implicitly refine the type within conditional blocks when using the `is` operator.
* MUST validate the invariant at runtime and throw a `CastException` upon failure when using the `as` operator.
* MUST bypass runtime validation and force compiler acceptance when using the `as!` operator.
* MUST validate the invariant at runtime and yield `Option<BaseType^Refinement>` when using the `as?` operator.

```C#
rawValue: Int = -10;

if rawValue is Int^Positive 
{
    ProcessAge(rawValue);
}

valid := rawValue as Int^Positive;

match rawValue as? Int^Positive 
{
    .Some(val) => ProcessAge(val),
    .None => Console.WriteLine("Invalid!")
}
```

## Mutability and Operations

* MUST implicitly downgrade the evaluation result of base type operators to the unrefined base type.
* MUST NOT implicitly assign an unrefined evaluation result to a mutable refined variable.
* MUST explicitly re-validate the invariant when assigning an unrefined result to a refined variable.

```C#
MutateAge(ref age: Int^Positive) 
{
    value: Int = age - 1;
    
    age = (age - 1) as Int^Positive; 
}
```
