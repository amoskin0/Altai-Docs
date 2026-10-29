# Command reference

[Contents](../README.md#english) · [Русский](../ru/cli.md)

`altai` below denotes the official command-line executable. In PowerShell, use `.\altai.exe` for an executable in the current directory. Quote paths containing spaces. Commands and compiler messages use English.

```text
altai tokens <source.alt>
altai tree <source.alt>
altai check <source.alt>
altai run <source.alt>
altai build [--target <name>] <source.alt> <output>
altai targets
altai help
```

| Command | Result |
| --- | --- |
| `tokens` | Lexical tokens with spans and positions; does not check the whole language semantics |
| `tree` | Parsed program tree; does not replace type/name checking |
| `check` | Full static validation, including all function bodies, return paths and resource limits; no program effects |
| `run` | Full validation followed by interpreted execution |
| `build` | Full validation followed by native executable generation; ordinary source expressions are not executed |
| `targets` | Registered targets and their implementation availability |
| `help` | Command usage |

Input uses strict UTF-8 and the [language specification](specification.md). A destination supplied to `build` must be new. Existing files are rejected. An explicit target selects the output platform, not a different language. The default is the compiler's host target; an unsupported default fails. Only `windows-x86_64` generation is implemented.

```powershell
& "C:\Altai Tools\altai.exe" run "C:\My Project\hello.alt"
```

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Success |
| 1 | Static source error |
| 2 | Tool or input/output error |
| 3 | Runtime arithmetic or byte-bounds error |

Static diagnostics include file, line, column, code, message, bounded source excerpt and a contextual hint where applicable. Invalid UTF-8 reports a byte offset. Runtime diagnostics do not include a source location and have these exact stderr lines:

```text
A0401: int64 overflow.
A0402: division by zero.
A0403: byte index out of bounds.
```

Each line ends with LF. A late runtime failure preserves earlier successful output. A static failure prevents all execution and executable creation. `check` and `build` accept ordinary `print(1 / 0)` because it is a runtime operation; `const x = 1 / 0` fails during checking. A stdout write failure stops execution with code 2; failed diagnostic writes preserve code 3 and may leave a partial diagnostic.

Program-level file opening, allocation and limit errors are values handled with `match`, not automatic process exit codes. `read_file` is executed only by `run` or a generated executable, never by `check` or `build`. Relative paths use the process working directory; file operations have ordinary OS authority and can block. See the [specification](specification.md) for error codes 1 through 7.
