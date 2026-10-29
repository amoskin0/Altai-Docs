# Native programs and targets

[Contents](../README.md#english) · [Русский](../ru/native.md)

`altai build source.alt program.exe` generates a console PE32+ AMD64 executable. Static validation and constant evaluation finish before file creation. Ordinary statements are compiled rather than executed by the builder. The executable runs without the source, Altai tool or C compiler.

## Targets

```text
altai build --target windows-x86_64 source.alt program.exe
altai targets
```

| Target | Format | ABI | Generation |
| --- | --- | --- | --- |
| windows-x86_64 | PE32+ | Microsoft x64 | Implemented |
| windows-aarch64 | PE32+ | Windows ARM64 | Unsupported |
| linux-x86_64 | ELF64 | System V AMD64 | Unsupported |
| linux-aarch64 | ELF64 | AAPCS64 | Unsupported |
| macos-x86_64 | Mach-O 64 | Darwin x86-64 | Unsupported |
| macos-aarch64 | Mach-O 64 | Darwin ARM64 | Unsupported |

The target registry separates OS, architecture, binary format and calling convention. A target is not inferred from the output filename. Unknown/unsupported targets fail before output creation. The default is the host target; selecting a target does not change the language. Compiler-host portability and generated-program support are distinct. A compiler cross-build does not establish execution on that host; a Windows PE file is not a Linux/macOS executable.

## Execution contract

Native output preserves the [specification's](specification.md) checked arithmetic, strict types, left-to-right effects, short-circuit logic, function calls, loops and runtime diagnostics. The builder does not pre-execute ordinary literal arithmetic or byte indexes. All branches and bodies are statically checked; only reached operations have runtime effects.

Source functions have isolated frames and real machine-code calls. Arguments are evaluated left to right before parameter copies. Opaque handles refer to immutable literal/dynamic bytes or mutable buffers. Assignment, arguments, returns and scope exits retain/release dynamic ownership; aliases stay valid while referenced. Buffer assignments share their contents; snapshot copies are independent. Length is a byte count. Indexing checks negative and upper bounds before reading the payload, then returns an unsigned byte as int64. Literal construction allocates no payload heap storage. Dynamic file input, buffers, growth and snapshots use the emitted heap runtime with automatic reference counting. The internal value/call representation is not an exported C ABI or a public pointer interface.

Loops reuse bounded storage without per-iteration stack growth. The checker rejects recursion and bounds cumulative modeled stack use to 64 KiB, including main, callees, return addresses and helper paths. Dynamic paths reserve an allowance of 352 bytes for two nested 168-byte helpers plus return addresses. Every frame is below 4096 bytes and touched on entry. This is not a general memory-management facility or a claim that all defects are impossible.

Literal/scalar executables import four KERNEL32.dll console/exit APIs. Dynamic executables additionally import GetProcessHeap, HeapAlloc, HeapFree, CreateFileW, ReadFile, CloseHandle, MultiByteToWideChar and GetLastError. Runtime helpers are emitted directly as machine code. It includes unwind metadata; code and immutable data use separate permissions, with no writable executable section. NX, ASLR and high-entropy virtual addressing are enabled. These properties do not remove OS/CPU dependencies or prove complete security. Native generation for `.alt` uses Altai's own emitter without an LLVM language backend, intermediate C or an external assembler/linker.

Dynamic storage limits are 16 MiB per object, 4096 objects and 64 MiB managed descriptor/payload capacity including growth peaks. Path conversion can use up to 65534 additional bytes. These limits do not bound RSS, compiler input, iterations or blocking I/O. File reads preserve arbitrary bytes, use OS/CWD path semantics and do not provide a sandbox.

## Output and errors

Output arguments are evaluated before stdout is accessed. Successful interpreted output is flushed. Earlier successful output survives a later arithmetic/bounds trap. Interpreter and native execution use the same exact A0401/A0402/A0403 messages and exit codes described in the [command reference](cli.md#exit-codes). A stdout failure exits with code 2; a failed runtime diagnostic preserves code 3.

On Windows, build writes a new adjacent temporary file, flushes and closes it, then publishes without replacing the destination. Existing output files, the source and hard links to it are rejected. Failures attempt cleanup; disk, power or filesystem failures can prevent cleanup, and absolute durability is not guaranteed. The output parent must exist and be writable.

The POSIX output writer has not been runtime-verified on Linux/macOS in the current Windows-based verification environment. It requires a trusted parent directory and does not guarantee directory fsync or crash durability. This does not add Linux/macOS native targets.
