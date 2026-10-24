# Packages and Visibility

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
// File: /Math/Vector.cat
Vector
{
    X: Int;
    Y: Int;
}

// File: /Math/Physics/Engine.cat
Engine
{
    Position: Vector; 
}
```

## Conditional Compilation Scopes and Manifest Flags

* MAY declare custom boolean feature flags and mutually exclusive flag groups within the package manifest (`package.toml`).
* MUST validate all file name suffixes across the package directory tree against the built-in target tags and the manifest's declared custom flags prior to compilation.

```toml
# MANIFEST: package.toml
[package]
name = "Engine"
os = ["windows", "linux"]
arch = ["amd64", "arm64"]

[flags]
render = ["vulkan", "dx12"]
features = ["steam", "telemetry"]
```

### Conditional Scope Narrowing and Coverage

* MUST assign every source file a conditional compilation scope defined by the set of suffix tags in its file name; an unsuffixed `.cat` file possesses the universal (unconstrained) scope.
* MUST permit a source file with conditional scope `CallerScope` to reference a declaration from a narrower conditional scope `TargetScope` if and only if `CallerScope` includes all suffix constraints of `TargetScope` (e.g., `App.windows.arm64.cat` MAY reference declarations from `App.windows.cat` and `App.cat`).
* MUST permit a broader or universal source file (e.g., `Console.cat`) to reference a conditionally compiled declaration if and only if an identical declaration signature is provided across all branches of the corresponding mutually exclusive target or flag group declared in the package manifest (e.g., implemented in both `Console.windows.cat` and `Console.linux.cat`).
* MUST reject compilation if a source file in a general or broader scope references a declaration that exists only in a narrower conditional scope without exhaustive coverage across the target group.

```C#
// File: Registry.windows.cat (Scope: {windows})
Win32Registry_ 
{
    ReadKey(path: String) -> String => "value";
}

// File: Console.windows.cat (Scope: {windows})
PlatformConsole_ 
{
    Write(text: String) { /* Win32 API */ }
}

// File: Console.linux.cat (Scope: {linux})
PlatformConsole_ 
{
    Write(text: String) { /* POSIX write */ }
}

// File: Console.cat (Universal Scope: {})
Console 
{
    WriteLine(text: String) 
    {
        // Valid: 'PlatformConsole_.Write' is defined across all supported OS targets
        PlatformConsole_.Write(text);

        // ERROR: 'Win32Registry_' is declared only in '.windows.cat' 
        // and cannot be referenced from the universal '.cat' scope
        key := Win32Registry_.ReadKey("HKLM");
    }
}
```

* MUST propagate private visibility (`_name`) downward from a broader conditional compilation file to all narrower conditional specialization files sharing the same `<BaseName>` (`<BaseName>.cat` -> `<BaseName>.windows.cat` -> `<BaseName>.windows.arm64.cat`) without requiring internal (`Name_`) visibility.

```C#
// File: Socket.cat (Universal Scope: {})
Socket 
{
    // Private field: hidden from other files in the package,
    // but accessible in 'Socket.windows.cat' and 'Socket.linux.cat'
    _rawDescriptor: Int;
    Host: String;
}

// File-scoped private helper in 'Socket.cat'
_socketBuffer 
{
    Size: Int;
}

// File: Socket.windows.cat (Narrower Scope: {windows}, same BaseName 'Socket')
+Socket 
{
    Connect(ref @) 
    {
        // Valid: accesses private '_rawDescriptor' and '_socketBuffer' from 'Socket.cat'
        buf := _socketBuffer { Size: 1024 };
        @._rawDescriptor = Win32.CreateSocket(buf.Size);
    }
}

// File: Other.windows.cat (Different BaseName 'Other')
Other 
{
    Test(s: Socket) 
    {
        // ERROR: '_rawDescriptor' is private to 'Socket.*.cat' files
        fd := s._rawDescriptor;
    }
}
```

## Visibility Rules

* MUST determine member access rights intrinsically by the identifier's lexical format.

### Public Members

* MUST allow global access from any file in any package that imports the declaring package.

### Internal Members

* MUST restrict access exclusively to files within the same compiled package.

### Private Members

* MUST restrict access to private members (`_name`) to the immediate enclosing lexical scope (`Data` type or file) and any narrower conditional compilation files sharing the exact same `<BaseName>` within the same directory.
* MUST allow a narrower conditional file (e.g., `Some.windows.cat` or `Some.windows.arm64.cat`) to access private fields, methods, and file-scoped private types (`_name`) declared in a broader conditional file of the same `<BaseName>` (e.g., `Some.cat` or `Some.windows.cat`).
* MUST NOT allow files with a different `<BaseName>` (e.g., `Other.cat` or `Other.windows.cat`) to access private members, even within the same package.

## Scope and Visibility Constraints

* MUST NOT expose restricted (private or internal) types within a public member's signature.

```C#
// PACKAGE: Geometry
// File: Shapes.cat
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
// File: Main.cat
use Geometry;

App
{
    Run() 
    {
        validCircle := Circle.New(5.0);
    }
}
```
