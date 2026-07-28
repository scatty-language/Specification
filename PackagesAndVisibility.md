# Packages and Visibility

Coggy organizes code into Packages. Files within the same package share a unified namespace and DO NOT require import directives to reference each other.

## Packages and Compilation Boundaries

```EBNF
UseDirective ::= "use" PackagePath ";"
PackagePath  ::= Identifier ("::" Identifier)*
```

The compilation unit in Coggy is the Package, defined by the presence of a manifest file (e.g., `package.toml`). The manifest explicitly declares the package's logical namespace, which MAY contain `::`, and its dependencies.

* **Boundary and Traversal**: The compiler recursively includes all `.cog` source files in the manifest's directory and all its subdirectories. Physical folder names MUST DO NOT affect the namespace.
* **Nested Packages**: If the compiler encounters a subdirectory containing another manifest file, it MUST halts traversal for that branch. That subdirectory is treated as an entirely isolated compilation unit.
* **Explicit Dependencies**: A package MUST explicitly declare any dependencies in its manifest (including local nested packages). A package CANNOT implicitly access another package simply because it resides in a subdirectory.
* **Unified Namespace**: All files compiled within the same package automatically resolve each other's public and internal members, completely ignoring their physical folder structure.
* **External Importing**: The `use` keyword brings the public members of an external (explicitly depended upon) package into the current file's namespace.
* **Ordering**: All `use` directives MUST appear at the top of the file, before any declarations.

```C#
// MANIFEST: package.toml (name = "Math")
// File: /Math/Vector.cog
Vector
{
    X: Int;
    Y: Int;
}

// File: /Math/Physics/Engine.cog
// No 'use' directive, 'Vector' is in the same compilation unit
// The '/Physics' folder is ignored
Engine
{
    Position: Vector; 
}
```

## Visibility Rules

Access rights for any member are intrinsically tied to its identifier's lexical format, as defined in the Identifiers section.

### Public Members

* **Access**: Available globally. Accessible from any file in any package that imports the declaring package.

### Internal Members

* **Access**: Available ONLY to files within the same compiled Package. Attempting to access an internal member from an external package is a strict compilation error.

### Private Members

* **Access**: Available ONLY within their immediate enclosing lexical scope.
  * If declared at the file level, they are private to that file.
  * If declared inside a `Data` type or extension, they are private to that block.

## Scope and Visibility Constraints

* **Exposure Restriction**: A public member MUST NOT expose a restricted (private or internal) type in its public signature (e.g., fields, return types, or parameters).
* **Variant Fields**: Variants defined within a variant field MUST use public identifiers, ensuring they are accessible wherever the field itself is accessible.

```C#
// PACKAGE: Geometry
// File: Shapes.cog
Circle
{
    Radius: Float;        // Public field
    _cache: Float;        // Private field

    New(radius: Float) -> @
    {
        return Circle { Radius: radius, _cache: 0.0 };
    }
}

Polygon_    // Internal Data
{
    Vertices: Int;
}

_helperData // Private Data (restricted to Shapes.cog)
{
    State: Int;
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
