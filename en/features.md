# Features and limits

[Contents](../README.md#english) · [Русский](../ru/features.md)

## Implemented

| Area | Current behavior |
| --- | --- |
| Source | `.alt`, strict UTF-8, optional initial BOM, LF/CRLF, case-sensitive names, `//` comments |
| Bindings | `var` mutable; `const` compile-time int64/bool; assignment updates only; fixed inferred types; lexical scopes without shadowing visible names |
| Arithmetic | int64 decimal literals, checked `+ - * / %`, unary signs, checked compound updates |
| Logic | bool, `!`, short-circuit `&&`/`||`, strict equality and signed integer comparisons |
| Control flow | `if`/`else if`/`else`, `while`, braces, optional grouped conditions |
| Functions | File-level `fn`, explicit int64/bool/bytes/buffer/result/status types, immutable value parameters, forward calls, `return`, checked return paths |
| Byte data | `bytes("literal")`, byte-counting `len`, checked indexing returning int64, immutable literals and dynamic bytes; mutable buffers, snapshots, raw file input, exhaustive result/status matching |
| Output | `print(expression)` for int64/bool or a string literal; one LF appended |
| Tools | Token/tree inspection, static checking, interpretation, own Windows x86-64 native generator |
| Diagnostics | Static source locations/excerpts/hints, stable codes, explicit runtime arithmetic/bounds failures |

The generated program is a standalone Windows console executable. The current compiler implementation uses C11 and is not self-hosting. The native generator emits AMD64/PE directly without an LLVM language backend, intermediate C, or an external assembler/linker for `.alt` compilation. The compiler's own build still needs a seed toolchain; generated programs depend on Windows system APIs.

## Bounds

| Resource | Limit |
| --- | --- |
| Total static bindings, including parameters and all declarations | 256 |
| Functions | 64 |
| Parameters or arguments per call | 16 |
| Active if/while/match blocks per body | 64 |
| Expression group/call/index/unary depth | 128 |
| Simultaneously live postfix operands | 128 |
| Cumulative modeled native call stack | 64 KiB |
| Dynamic object length/capacity | 16 MiB |
| Live dynamic objects | 4096 |
| Managed descriptor/payload capacity including growth peak | 64 MiB |

Loops have no fixed iteration limit and do not consume a new binding slot each iteration. Each native frame is below 4096 bytes and is touched on entry. The stack bound is a generated-code resource model, not a measurement of the interpreter's exact stack usage. See the [specification](specification.md) for counting and error rules.

## Not implemented

Floating point, string variables, arrays, arbitrary indexed byte mutation, byte slicing/equality, `for`, `break`, `continue`, recursion, void functions, discarded calls, modules, packages, closures, overloads, foreign-function interfaces and a general memory model are not implemented. There are no browser APIs or general application frameworks in this subset. Linux/macOS native generation is not implemented.

The byte classifiers written in Altai are helpers; complete source validation, tokenization and decoded-string storage remain in the compiler implementation. File input and buffers provide data access; the complete lexer and compiler are still implemented in C11. Runtime storage caps do not bound compiler allocations, total RSS, iterations or blocking I/O. Programs use the OS filesystem authority of the running account.
