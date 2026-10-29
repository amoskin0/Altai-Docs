# Getting started

[Contents](../README.md#english) · [Русский](../ru/getting-started.md)

Altai source files use the `.alt` extension and UTF-8. Keywords use English. These instructions describe the current Windows x64 command-line tool. They do not require building the compiler from source.

## Write a program

Save the following as `hello.alt` in a directory where you can create files:

```text
print("Hello, world!")
```

With the official command-line tool available in that directory, open PowerShell there:

```powershell
.\altai.exe check .\hello.alt
.\altai.exe run .\hello.alt
```

`check` validates the entire program without running it. `run` validates it first, then interprets it. The program prints `Hello, world!` followed by a newline. An error in a later statement can prevent execution of the entire program if it is a static error.

## Build a standalone executable

```powershell
.\altai.exe build --target windows-x86_64 .\hello.alt .\hello.exe
.\hello.exe
```

Choose a new output path: `build` does not replace existing files. It checks the program but does not execute ordinary expressions. A generated executable runs on Windows without the source file, Altai tool or a C compiler. Only Windows x86-64 native output is implemented. See [targets and native behavior](native.md).

## Variables, constants and functions

```text
const quantity = 3
var price = 120
price += 5
print(price * quantity)

fn is_digit(code: int64) -> bool {
    if code < 48 {
        return false
    }
    return code <= 57
}

print(is_digit(53))
```

The output is `375` then `true`. `var` declares an initialized mutable binding. `const` declares an integer/boolean constant evaluated during checking. `=` only updates an existing mutable binding. Types are fixed, and conditions require bool. Function parameter/result types are explicit; parameter values are immutable. Calls require parentheses. Statements use newlines, without semicolons.

## Loops and bytes

```text
var data = bytes("Altai")
var index = 0
var total = 0
while index < len(data) {
    total += data[index]
    index += 1
}
print(total)
```

This prints `491`. `len` counts UTF-8 bytes rather than characters. An indexed byte is an int64 from 0 through 255. Indexes are checked at runtime. Byte contents are immutable; mutable buffers and raw file input are described below. The [specification](specification.md) defines evaluation order, errors and resource limits.

## Altai Editor

Altai Editor can open, edit and save `.alt` files. F5 saves the active document and runs the interpreter; Shift+F5 stops it. F5 does not invoke native build. Its interface supports English and Russian. Source keywords and compiler diagnostics remain English.

See [command reference](cli.md) for inspecting tokens and trees, and [features and limits](features.md) for the supported subset.

## Files and buffers

`read_file(bytes("input.alt"), 1048576)` returns `result<bytes>`: `ok(source)` exposes raw immutable bytes; `err(code)` exposes an int64 error. File contents preserve NUL and invalid UTF-8. The path itself must be valid, nonempty UTF-8 without NUL. Relative paths use the running process's working directory and ordinary OS permissions.

```text
match read_file(bytes("input.alt"), 1048576) {
    ok(source) { print(len(source)) }
    err(code) { print(code) }
}
```

`buffer(limit)` returns `result<buffer>`. `append_byte(output, value)` returns `status`; match it using `ok { ... }` and `err(code) { ... }`. `snapshot(output)` returns `result<bytes>` with an independent immutable copy. Buffer assignments share storage; byte snapshots do not change when that buffer changes. `len` and checked indexing accept both bytes and buffer.

```text
match buffer(2) {
    ok(output) {
        match append_byte(output, 65) {
            ok {
                match snapshot(output) {
                    ok(copy) { print(copy[0]) }
                    err(code) { print(code) }
                }
            }
            err(code) { print(code) }
        }
    }
    err(code) { print(code) }
}
```

This buffer example prints `65`. Memory is released automatically. Both arms of a match are required, in `ok` then `err` order. The [specification](specification.md#files-buffers-and-results) defines types, error codes and ownership.

Managed storage is limited to 16 MiB per object, 4096 live objects and 64 MiB of descriptors/payload capacity including growth peaks. These are not whole-process memory or time limits. File access is not a sandbox and has no deadline.
