# Packages and Visibility

Coggy organizes code into Packages. Files within the same package share a unified namespace and DO NOT require import directives to reference each other.

## Packages and Imports

The compilation unit is the Package. Source code files strictly act as physical containers contributing to the Package's global namespace.

```EBNF
UseDirective ::= "use" PackagePath ";"
PackagePath  ::= Identifier ("::" Identifier)*
```

* **Unified Namespace**: All files compiled within the same package automatically resolve each other's public and internal members.
* **External Importing**: The `use` keyword strictly brings the public members of an external package into the current file's namespace.
* **Ordering**: All `use` directives MUST appear at the top of the file, strictly before any declarations.

```C#
// PACKAGE: Math
// File: Vector.cog
Vector
{
    X: int;
    Y: int;
}

// File: Engine.cog
// No 'use' directive needed! 'Vector' is in the same package
Engine
{
    Position: Vector; 
}
```

## Visibility Rules

Access rights for any member are intrinsically tied to its identifier's casing and format.

```EBNF
PublicId   ::= [A-Z] [a-zA-Z0-9]* (but not ending with "_")
InternalId ::= [A-Z] [a-zA-Z0-9]* "_"
PrivateId  ::= "_" [a-z] [a-zA-Z0-9]*
```

### Public Members

Members starting with an uppercase letter are **Public**.

* **Access**: Available globally. Accessible from any file in any package that imports the declaring package.

### Internal Members

Members starting with an uppercase letter and strictly ending with a single underscore `_` are **Internal**.

* **Access**: Available ONLY to files within the same compiled Package. Attempting to access an internal member from an external package is a strict compilation error.

### Private Members

Members starting with an underscore `_` followed by a lowercase letter are **Private**.

* **Access**: Available ONLY within their immediate enclosing lexical scope. 
  * If declared at the file level, they are strictly private to that file.
  * If declared inside a `Data` type or extension, they are strictly private to that block.

## Scope and Visibility Constraints

* **Exposure Restriction**: A public member MUST NOT expose a restricted (private or internal) type in its public signature.
* **Enum Variants**: Variants within a public `enum` MUST strictly be public.

```C#
// PACKAGE: Geometry
// File: Shapes.cog
Circle
{
    Radius: float;        // Public field
    _cache: float;        // Private field

    New(radius: float) -> @
    {
        return Circle { radius, radius };
    }
}

Polygon_    // Internal Data
{
    Vertices: int;
}

_helperData // Private Data (restricted to Shapes.cog)
{
    State: int;
}

InvalidData
{
    // ERROR: Public type 'InvalidData' cannot expose private type '_helperData'
    Field: _helperData;   
}


// PACKAGE: App
// File: Main.cog
use Geometry; // Imports the external package

App
{
    Run() 
    {
        // ERROR: '_cache' is private to 'Circle'
        circle := Circle { Radius: 5.0, _cache: 0.0 }; 
        
        // Valid: Public factory method inside a public type
        validCircle := Circle.New(5.0);
        
        // ERROR: 'Polygon_' is internal to Package 'Geometry'
        poly := Polygon_ { }; 
    }
}
```
