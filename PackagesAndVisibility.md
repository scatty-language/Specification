# Packages and Visibility

```EBNF
UseDirective ::= "use" PackagePath ";"
PackagePath  ::= Identifier ("::" Identifier)*
```

* MUST organize code into Packages acting as isolated compilation units.
* MUST automatically resolve public and internal members across all files within the same package without import directives.

## Packages and Compilation Boundaries

* MUST define a Package via the presence of a manifest file.
* MAY include `::` within the package's logical namespace declared in the manifest.
* MUST recursively include all source files within the manifest's directory and subdirectories.
* MUST NOT alter the logical namespace based on physical directory structures.
* MUST halt directory traversal upon encountering a nested manifest file, treating that subdirectory as an isolated compilation unit.
* MUST explicitly declare all package dependencies (including nested packages) within the manifest.
* MUST NOT implicitly access a nested package.
* MUST import public members of an explicitly depended-upon external package into the current file's namespace using the `use` directive.
* MUST place all `use` directives at the file's top level, prior to any declarations.

```C#
// MANIFEST: package.toml (name = "Math")
// File: /Math/Vector.cog
Vector
{
    X: Int;
    Y: Int;
}

// File: /Math/Physics/Engine.cog
Engine
{
    Position: Vector; 
}
```

## Visibility Rules

* MUST determine member access rights intrinsically by the identifier's lexical format.

### Public Members

* MUST allow global access from any file in any package that imports the declaring package.

### Internal Members

* MUST restrict access exclusively to files within the same compiled package.

### Private Members

* MUST restrict access exclusively to the immediate enclosing lexical scope (e.g., the declaring file, `Data` type, or extension block).

## Scope and Visibility Constraints

* MUST NOT expose restricted (private or internal) types within a public member's signature.

```C#
// PACKAGE: Geometry
// File: Shapes.cog
Circle
{
    Radius: Float;
    _cache: Float;

    New(radius: Float) -> @
    {
        return Circle { Radius: radius, _cache: 0.0 };
    }
}

Polygon_ 
{
    Vertices: Int;
}

_helperData
{
    State: Int;
}


// PACKAGE: App
// File: Main.cog
use Geometry;

App
{
    Run() 
    {
        validCircle := Circle.New(5.0);
    }
}
```
