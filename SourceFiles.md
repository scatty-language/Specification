# Source Files

* MUST use the `.cat` file extension for all source code files.
* MUST ignore files without the `.cat` extension during package traversal.
* MUST encode source files in UTF-8.
* MUST NOT contain a Byte Order Mark (BOM).
* MUST reject source files containing a UTF-8 BOM (`U+FEFF`) at the absolute beginning of the file.
* MUST reject source files containing invalid or malformed UTF-8 byte sequences.

## Whitespace

* MUST accept both Line Feed (`LF`, `\n`) and Carriage Return + Line Feed (`CRLF`, `\r\n`) as valid line terminators.
* MAY use spaces (`U+0020`), horizontal tabs (`U+0009`), and line terminators to separate tokens.
* MUST NOT assign semantic meaning to whitespace outside of `String` literals.

## Conditional Compilation Suffixes

* MUST express all conditional compilation rules exclusively via dot-separated suffixes in source file names prior to the `.cat` extension.
* MUST NOT support in-code preprocessor directives or conditional compilation macros within `.cat` source files.
* MUST enforce the following canonical suffix order in source file names:
  `<BaseName>[.<custom>][.<mode>][.<os>][.<arch>].cat`
* MUST reject any source file whose suffixes violate the canonical category order (`<custom>` followed by `<mode>`, followed by `<os>`, followed by `<arch>`).
* MUST recognize the following built-in conditional suffix identifiers:
  * Build Mode (`<mode>`): `debug`, `release`
  * Operating System (`<os>`): `windows`, `linux`, `macos`, `ios`, `android`, `freebsd`, `wasm`
  * Architecture (`<arch>`): `amd64`, `arm64`, `x86`, `arm`, `riscv64`, `wasm32`
* MAY include one or more custom flag suffixes (`<custom>`) declared in the enclosing package's manifest file.
* MUST reject any source file containing an unrecognized suffix identifier that is neither a built-in identifier nor a custom flag declared in the package manifest.
* MUST include a suffixed source file in the active compilation unit if and only if every suffix tag specified in its file name matches the active build configuration.

```text
// Valid canonical suffix ordering: <BaseName>[.<custom>][.<mode>][.<os>][.<arch>].cat
Console.cat
Console.windows.cat
Console.windows.arm64.cat
Console.debug.windows.arm64.cat
Renderer.vulkan.release.windows.amd64.cat

// Invalid: violates canonical order (<os> placed before <mode> or <arch> before <os>)
Console.windows.debug.cat       // ERROR: <mode> MUST precede <os>
Console.arm64.windows.cat       // ERROR: <os> MUST precede <arch>
Renderer.windows.vulkan.cat     // ERROR: <custom> MUST precede <os>
```
