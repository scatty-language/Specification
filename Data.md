# Data

```EBNF
DataDecl         ::= Identifier ["<" TypeArgs ">"] [":" Interfaces] [":::" TagType] ( "{" Member* "}" | "(" [ShortFieldList] ")" ";" )

Interfaces       ::= TypeIdentifier ("," TypeIdentifier)*
TagType          ::= Type | "none"
 
Member           ::= VariantDecl | FieldDecl | InlineFieldDecl | ConstDecl | PropertyDecl | MethodDecl

VariantDecl      ::= "::" PublicId ["(" PayloadFieldList ")"] ["=" Expr] [":::" Expr] ";"
PayloadFieldList ::= PayloadField ("," PayloadField)*
PayloadField     ::= [FieldModifier] [Identifier ":"] Type ["=" Expr] [":::" Decimal]
 
ShortFieldList   ::= ShortField ("," ShortField)*
ShortField       ::= [FieldModifier] (Identifier | "_") ":" Type ["=" Expr] [":::" Decimal]
 
FieldDecl        ::= ShortField ";"
InlineFieldDecl  ::= "@" ShortField ";"
FieldModifier    ::= "$" | "#" | "#$"
```

* MUST be declared at the file's top level.
* MUST evaluate default values prior to initializer execution.
* MUST determine type equality via structural comparison of identity fields, or all fields if no identity fields exist.
* MUST lay out fields sequentially in memory.
* MAY apply memory padding to fields.
* MUST occupy a minimum of 1 byte.

```C#
Vector 
{
    $Id: Int;
    #X: Int;
    #Y: Int = 1;
}

Vector($Id: Int, #X: Int, #Y: Int = 1);
```

## Padding and Bitfields

### Anonymous Padding Fields

// TODO: add ignoring in soa arrays

* MAY declare anonymous padding fields using the wildcard (`_`).
* MUST NOT be read, mutated, or accessed.
* MUST NOT be required during structural initialization.
* MUST NOT declare default values.
* MUST NOT use declarative modifiers.

### Bitfields

* MAY define exact field bit-width using the Physical Layout operator (`:::`).
* MUST pack adjacent bitfields into the smallest contiguous memory boundary.
* MUST apply only to scalar integer primitives and `Bool` fields.
* MUST NOT be passed by reference.

```C#
NetworkHeader 
{
    Version: Byte;
    
    _: Byte; 
    _: Byte;
    _: Byte;
    
    IsSYN: Bool ::: 1;
    IsACK: Bool ::: 1;
    _: Byte     ::: 6;
}
```

## Variants

* MAY declare mutually exclusive variants alongside standard fields.
* MUST encapsulate a hidden discriminant tag if variants are declared.
* MUST declare at least two variants if any variant is declared.
* MUST allocate standard fields in memory independent of the active variant.
* MAY declare variant-specific payloads.
* MAY omit payload field identifiers.

```C#
Option<T> 
{
    ::None;
    ::Some(T);

    Unwrap(@) throws -> T 
    {
        if @ is .Some(val) => return val;
        throw Exception.New("Unwrapped None");
    }
}
```

### Explicit Tags

* MUST default the implicit variant tag type to `Int` with sequential values starting at `0`.
* MAY explicitly define the tag type and individual tag values.
* MUST resolve the explicit tag type to a scalar integer primitive.
* MUST evaluate explicit variant tag values as compile-time constants.
* MUST assign an omitted variant tag the value of the preceding variant's tag plus `1`.

```C#
HttpResponse : ~Printable ::: Int 
{
    // Variants
    ::Success ::: 200;
    ::NotFound ::: 404;
    ::Error(code: Int) ::: 500;

    RequestUrl: String;

    Print(@) => Console.WriteLine("\([@.RequestUrl]) failed");
}
```

### Untagged Unions

* MAY remove the discriminant tag by explicitly declaring `::: none`.
* MUST NOT use safe pattern matching operators (`is`, `as`, `as?`, `match`) on untagged unions.
* MUST extract payloads using the unchecked cast operator (`as!`).

```C#
FloatIntUnion ::: none 
{
    ::FloatBits(Float);
    ::IntBits(Int);
}

myUnion := FloatIntUnion::IntBits(42);

floatVal := myUnion as! .FloatBits;
```
