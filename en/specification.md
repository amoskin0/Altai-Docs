# Altai language specification

## Source and tokens

Source files use `.alt` and strict UTF-8. A single initial BOM is accepted: it occupies bytes but not a source column. Other U+FEFF characters are not discarded. Validation checks the entire UTF-8 input, then all CR characters, then lexical structure. A lone CR is invalid everywhere, including strings and comments. LF and CRLF each advance one line. Lines and columns are one-based; columns count Unicode scalar values, with a tab occupying one column.

Token spans `[start, end)` are zero-based half-open byte ranges. The end token has an empty span at the input length and the position after the last scalar. After a final LF/CRLF it is at column one of the next line.

Whitespace outside strings/comments consists of ASCII space, tab, LF, and CRLF. `//` starts a comment ending at the physical newline or end of input. Block comments are unsupported. Names start with ASCII letters, Russian Cyrillic letters including U+0401/U+0451, or `_`; subsequent characters may also contain ASCII digits. Names are case-sensitive and not normalized. Other Unicode letters and combining marks are unsupported in names. Visually similar Latin and Cyrillic letters remain distinct.

`const`, `var`, `true`, `false`, `if`, `else`, `while`, `fn`, `return` and `match` are reserved keywords. `print`, `bytes`, `len`, `read_file`, `buffer`, `append_byte` and `snapshot` are reserved built-in names and cannot be declared. `ok`/`err` and type names are contextual. `let`, `Var`, `variable`, and former localized names are ordinary user names, not declaration forms or aliases. Number tokens contain ASCII digits. Leading zeroes remain decimal. Suffixes, separators, exponents, and fractional literals are unsupported. `->`, `&&`, `||`, the two-character comparisons and compound assignments are indivisible tokens; `:` separates a function parameter name from its type. These symbols are accepted only in their grammatical contexts. The former `:=` declaration is rejected.

Strings use double quotes. Escapes are `\"`, `\\`, `\n`, `\r`, and `\t`. Unescaped LF, CR, C0/C1 controls, and U+007F are rejected. An escaped CR is value data, not a source newline. U+2028/U+2029 are permitted string data. Values contain decoded UTF-8 bytes. There is no NUL escape.

## Grammar

```text
program     = { NEWLINE | statement | function } END
statement   = (simple | conditional | loop | matching | returning) terminator
simple      = ("var" | "const") NAME "=" expression
            | NAME assignment expression
            | "print" "(" expression ")"
assignment  = "=" | "+=" | "-=" | "*=" | "/=" | "%="
conditional = "if" expression lines block
              [ lines "else" lines (conditional | block) ]
loop        = "while" expression lines block
function    = "fn" NAME "(" [ parameters ] ")" "->" type lines block terminator
parameters  = NAME ":" type { "," NAME ":" type }
type        = "int64" | "bool" | "bytes" | "buffer" | "status"
            | "result" "<" ("bytes" | "buffer") ">"
matching    = "match" expression lines "{" lines success lines
              "err" "(" NAME ")" lines block lines "}"
success     = "ok" "(" NAME ")" lines block | "ok" lines block
returning   = "return" expression
block       = "{" { NEWLINE | statement } "}"
lines       = { NEWLINE }
expression  = disjunction
disjunction = conjunction { "||" conjunction }
conjunction = equality { "&&" equality }
equality    = ordering { ("==" | "!=") ordering }
ordering    = sum { ("<" | "<=" | ">" | ">=") sum }
sum         = product { ("+" | "-") product }
product     = prefix { ("*" | "/" | "%") prefix }
prefix      = ("+" | "-" | "!") prefix | postfix
postfix     = primary { "[" expression "]" }
primary     = NUMBER | STRING | NAME | "true" | "false"
            | NAME "(" [ expression { "," expression } ] ")"
            | "(" expression ")"
```

This grammar omits expression-internal newline handling for readability. Newlines inside expression/call parentheses or index brackets are whitespace. Outside open delimiters a newline before `[` ends the completed statement. Outside them, newlines immediately after a binary operator continue the unfinished expression; a newline before an operator ends the statement. For example:

```text
var total = 100 +
    20 +
    5
print(total)
```

The output is `125`. A unary prefix does not itself permit an ungrouped newline. Neither does an assignment operator: an initializer or assignment RHS starts on the same line as `=` or `+= -= *= /= %=` unless its opening parenthesis is on that line. Parameter names, `:` and types stay on one line, as do `->` and the result type. Comments may end a continued binary line. Newlines may separate an if/while/function header from its opening brace. Examples use the opening brace on the header line; there is no formatter command.

A terminator consumes a newline or recognizes an immediately following closing brace/end token without consuming it. Empty lines/comments and a missing final newline are allowed. A closing brace terminates the final statement of its block, allowing inline blocks. A completed conditional or loop still needs a newline, end of file or enclosing closing brace before a following statement. A semicolon is tokenized for diagnostics but rejected by the grammar.

Calls always use parentheses, with whitespace before `(` allowed. `print(1 + 2)` outputs `3`; `print 1 + 2` and `print(1 + 2) * 3` are rejected. Output takes exactly one argument, cannot be nested as an expression and has no usable result. Call delimiters count toward expression nesting; function signature delimiters do not. Arbitrary discarded value-returning call statements are unsupported.

## Names and types

```text
const quantity = 3
var price = 120
price = price + 5
print(price * quantity)
```

`var name = expression` declares a new mutable value with mandatory initialization. `name = expression` only assigns to an existing visible mutable value; it never declares one. An unknown assignment destination gives A0301, so a misspelling cannot silently create another binding. During name/type checking, destination-name errors precede semantic errors in the right-hand expression. Whole-input lexical validation and parsing still finish before this pass.

A name becomes visible only after its initializer: `var x = x + 1` gives A0301, while `var x = 1` followed by `x = x + 1` is valid. `var` and `const` cannot redeclare or shadow any visible binding; duplicates and built-in names give A0302. Uninitialized declarations, explicit binding type annotations, and `var name := expression` are not supported.

`const name = expression` declares an immutable constant evaluated while checking/building. It may use integer/boolean literals, earlier constants and supported scalar operators. Calls and runtime bindings, including immutable function parameters, are forbidden even in a skipped short-circuit operand (A0306). Invalid arithmetic in an evaluated operand also gives A0306; arithmetic in a skipped operand does not execute. Repeated declarations for a visible name give A0302; assignment to a constant gives A0305. There is no `:=` alias or runtime-immutable `let` declaration.

Typing is static and strict. Value types are `int64`, `bool`, immutable `bytes`, mutable `buffer`, `result<bytes>`, `result<buffer>` and `status`. A local or file binding's type is inferred at declaration and preserved on assignment; function parameter/result types are explicit. String initializers/assignments give A0303. There are no implicit conversions, explicit binding type annotations, other numeric types, or string variables/constants. The file and each brace-delimited block have lexical scopes, without forward visibility or shadowing of visible names. New declarations, parameters and match arm bindings consume the binding limit.

Errors: A0301 unknown name; A0302 duplicate/reserved declaration; A0303 type error; A0304 out-of-range literal; A0305 assignment to an immutable binding; A0306 invalid constant initializer. `print (("text"))` is valid; string arithmetic and string declarations are type errors.

Compound assignments `+= -= *= /= %=` require an existing mutable int64 destination and an int64 RHS. They never declare a binding or work on bool. Destination-name/mutability errors precede RHS semantic errors. At runtime the old destination is read first, the RHS evaluates once, then the checked arithmetic result is stored. An RHS error or arithmetic trap prevents the store and preserves earlier effects. These operators use the same overflow, zero-divisor and MIN/-1 rules as ordinary arithmetic. There are no increment/decrement operators.

## Arithmetic and output

The range is -9223372036854775808 through 9223372036854775807. Magnitude 9223372036854775808 is allowed only directly after unary minus, with intervening whitespace permitted. `-(9223372036854775808)` contains an invalid positive literal. `--9223372036854775808` parses, but its outer negation overflows: a runtime error in an ordinary statement, A0306 in a constant initializer.

Precedence from highest to lowest is grouping/calls/indexing, unary `+ - !`, `* / %`, `+ -`, ordered comparisons, equality, `&&`, then `||`. Binary operators associate left. Operands and statements evaluate left to right. Division truncates toward zero; remainder has the dividend's sign. A zero divisor gives A0402. Overflow in addition, subtraction, multiplication, negation, MIN/-1, and MIN%-1 gives A0401. Checks precede dangerous machine operations; values do not silently wrap.

Ordinary expressions retain runtime checks even when containing only literals. `check` and `build` accept `print(1/0)`; execution exits with code 3. Constant initialization is the exception: `const x = 1/0` is a static error before any output.

`print(expression)` accepts one integer/boolean expression or string literal and appends LF. The required argument parentheses can contain additional grouping. Booleans print exactly `true` or `false`. Integers use decimal output without locale formatting. The argument is evaluated before stdout is accessed. Every successful interpreted output is flushed. A late runtime error preserves earlier output and prevents later statements.

## Read-only bytes

```text
var data = bytes("abc")
print(len(data))
print(data[0])
```

This prints `3` and `97`. `bytes` requires exactly one string literal expression, including grouped literals. It exposes the literal's decoded UTF-8 bytes without a trailing terminator. `len` requires one bytes or buffer value and returns its byte count as int64. A non-ASCII character can occupy multiple bytes. Existing string escapes retain their meaning; byte construction does not add escapes or allow malformed UTF-8 or embedded NUL in source literals. Raw file input and buffers can contain those byte values.

`base[index]` evaluates the base once, then the index once, left to right. The base must be bytes or buffer and the index int64. The result is an int64 from 0 through 255, with no sign extension. Both negative indexes and indexes greater than or equal to the byte length produce A0403 before memory access. Empty values have length zero and every index fails. Known bad indexes in ordinary expressions are runtime errors when reached; `check` and `build` do not execute them. Skipped branches and short-circuit operands perform no access, while all names and types still receive static checks.

Calls and grouped expressions may supply the base: `get_data()[0]` and `(data)[0]`. Postfix indexing binds tighter than unary or binary operators. Each index's brackets count toward the expression-depth bound and permit internal newlines. A second index on the int64 result is a type error. Indexing cannot be an assignment destination.

Bytes contents are immutable. A bytes `var` can be reassigned another bytes value; assignment and function calls preserve other references. Bytes parameters are immutable bindings, and functions may return literals, parameters or dynamic bytes. Literal storage remains valid throughout execution; dynamically produced bytes remain valid while referenced. Generated literals use read-only program data without payload allocation. Dynamic values use the managed runtime described below. The internal value representation is not a public pointer or foreign-function ABI.

`bytes`, `len`, `read_file`, `buffer`, `append_byte` and `snapshot` are reserved built-in names, including in function/parameter declarations. All calls remain forbidden in `const` initializers. Bytes/buffers have no equality, ordering, arithmetic, pointer conversion, slicing or raw-byte output. `print(data)` is a type error; lengths and indexed bytes can be printed as int64.

## Files, buffers and results

| Operation | Parameters | Result |
| --- | --- | --- |
| `read_file(path, limit)` | bytes path, int64 byte limit | `result<bytes>` |
| `buffer(limit)` | int64 byte limit | `result<buffer>` |
| `append_byte(target, value)` | buffer, int64 byte from 0 through 255 | `status` |
| `snapshot(target)` | buffer | `result<bytes>` |

`read_file` returns raw file bytes, including all 256 byte values. It does not validate file contents as UTF-8, remove a BOM, normalize CRLF, append a terminator or stop at NUL. The path itself must be nonempty valid UTF-8 without NUL, at most 32767 bytes and at most 32766 UTF-16 units. Relative paths use the process working directory, which need not be the source directory. Absolute and operating-system namespace paths use the account and filesystem authority of the interpreter or generated program. There is no project-root sandbox, regular-file-only rule or read timeout.

The limit is an upper bound on returned byte length, not a request to return a truncated prefix. An empty file succeeds with limit zero. A file exceeding the limit gives error 5 and no partial byte value. Storage grows geometrically as bytes arrive; the entire requested limit is not allocated before reading. A final one-byte probe distinguishes exact-limit EOF from an oversized input. Reads can observe concurrent file changes and do not promise an atomic snapshot of the filesystem object.

`buffer(limit)` creates an empty mutable buffer. A nonnegative zero limit is valid. `append_byte` appends one byte when capacity and runtime resources permit; a rejected byte or exhausted limit leaves its contents unchanged. A value outside 0..255 gives error 7. `len` and read-only indexing work on both bytes and buffer; direct element assignment is unavailable. Copying, passing or returning a buffer aliases the same stable object, including after storage growth. An immutable parameter or match binding prevents rebinding, not mutation through `append_byte`. `snapshot` copies the current contents into independent immutable bytes, so later buffer changes do not change the copy. Copying a bytes binding shares immutable contents rather than making a payload copy.

Dynamic bytes, buffers and successful result payloads use automatic reference counting. References remain valid through rebinding, function calls/returns and scope exits while another reference exists. Storage is reclaimed after its last reference is released; execution cleanup also reclaims remaining tracked objects after a fatal exit. Literal bytes keep program lifetime. There are no exposed addresses, explicit free operations or general-purpose allocation API.

A `result<bytes>` or `result<buffer>` contains a success payload or an integer error code. A `status` contains success without payload or an integer error code. Results/status values can be stored, assigned with the same type, passed to functions and returned. They cannot be used as booleans, indexed, printed or implicitly unwrapped. User-written success/error constructors, other result payload types, nested results and exception syntax are unavailable.

```text
match read_file(bytes("input.bin"), 1048576) {
    ok(data) {
        print(len(data))
    }
    err(code) {
        print(code)
    }
}

match buffer(2) {
    ok(output) {
        match append_byte(output, 255) {
            ok { print(output[0]) }
            err(code) { print(code) }
        }
        match snapshot(output) {
            ok(copy) { print(len(copy)) }
            err(code) { print(code) }
        }
    }
    err(code) { print(code) }
}
```

`match expression { ... }` evaluates its result/status subject once. Every match has exactly two arms in this order: `ok(name) { ... }` for a result payload, or `ok { ... }` for status, followed by `err(code) { ... }`. Both arms are mandatory, even if empty. There is no wildcard or additional pattern syntax. All arms are statically checked, but only the selected arm executes. Payload and error-code bindings are immutable and visible only within their own arm; they cannot shadow a visible name. An error code has type int64. Each match adds one control-depth level shared with if/while; each arm binding counts toward the total static binding limit. Both returning arms can satisfy a function return path. `ok` and `err` are contextual arm names rather than globally reserved keywords.

These recoverable operation codes are values handled by `match`, not process exit codes or A0401/A0402/A0403 traps:

| Code | Meaning |
| --- | --- |
| 1 | Invalid argument: malformed/empty/NUL-containing/oversized path or negative limit |
| 2 | File/path not found |
| 3 | Access or sharing denied |
| 4 | Other open/read/close I/O failure |
| 5 | Object length, live storage or live object-count limit exceeded |
| 6 | Memory allocation failed |
| 7 | Appended byte outside 0..255 |

A handled error alone does not stop execution or set a nonzero process exit code. Arithmetic or index traps while evaluating operation arguments still use the ordinary fatal runtime rules. Paths and negative limits are validated before opening files; a limit above the object maximum gives error 5. `append_byte` checks the byte range before growth/limit checks.

Managed limits are 16 MiB (16777216 bytes) per dynamic object, 64 MiB (67108864 bytes) of live descriptor-plus-payload storage, and 4096 live dynamic objects. Descriptors occupy 64 bytes each. Capacity counts, not just used length; growth checks the peak with old and replacement payloads both live, and a snapshot needs space for its independent copy. Released objects return their storage/count budget. These are not caps on total process RSS, allocator/OS overhead, source buffers, compiler intermediate data, executable images, path staging, time, loop iterations or total I/O. A Windows path conversion can temporarily allocate up to 65534 additional bytes outside managed accounting.

## Booleans, comparisons, and conditions

`true` and `false` have type `bool`. Arithmetic/unary signs require `int64`. Ordered comparisons `< <= > >=` require two integers and produce bool using signed order. Equality `== !=` accepts two values of the same scalar type: int64/int64 or bool/bool. Strings cannot be compared. Bool and int64 never convert implicitly.

Comparison/equality operators associate left at their respective precedence. `1 < 2 == true` means `(1 < 2) == true`. Python-style chained comparisons are unsupported: `1 < 2 < 3` fails because its second comparison receives a boolean operand. `!` requires bool and negates it. `&&` and `||` require bool operands and evaluate left to right with short circuit: a false left side of AND or true left side of OR skips the entire right side, including calls, output and arithmetic traps. Both sides are still fully parsed and statically checked, so `false && unknown` is an error. A constant `false && (1 / 0 == 0)` is valid, but `false && runtime_value` is not a valid constant initializer.

```text
var price = 120
var expensive = price > 100
if expensive {
    price = price - 20
    print(price)
} else {
    print(0)
}
print(true == false)
```

The condition must be bool. Parentheses around it are optional expression grouping; braces delimit blocks and indentation has no grammatical meaning. The opening brace may follow the condition on the same line or after newlines/comments. No `end` keyword is used. `else`, when present, follows the closing then-brace, with intervening newlines/comments allowed, and introduces another block or `if` condition. Newlines after `else` are allowed. An `else if` chain evaluates conditions in order until the first true branch, otherwise its final `else` if present. Each branch has a separate lexical scope; the chain consumes the same bounded control-depth resources as nested conditionals.

Both branches are fully parsed and checked before any execution/output, including invalid constant initializers in an unselected branch. At run time the condition is evaluated once, then only the selected branch executes. A runtime error in an unselected branch does not occur. Earlier successful output and checked-arithmetic error ordering are preserved.

Inside a block, `=` updates an already visible mutable binding. `var` and `const` declare new block-local bindings and reject names already visible in any enclosing scope. Local names disappear at the closing brace. Sibling blocks may independently declare the same spelling using distinct slots. A name introduced only inside a branch is not available afterward, even if both branches use that spelling. All bindings are initialized before use; every declaration in both branches counts toward the total limit. Assignments do not consume new binding slots.

## While loops

```text
var count = 0
var total = 0
while count < 5 {
    var next = count + 1
    total = total + next
    count = next
}
print(total)
```

This program prints `15`. The condition must be bool and is evaluated before every iteration, including the first. A false condition skips the body. Parentheses are optional grouping; the opening brace may follow on the header line or after newlines/comments. No `end` is used. `while` has no `else` clause.

The complete body is parsed and checked before execution, even for `while false`. Invalid names, types, or constant initializers in an unreachable body are static errors. Runtime errors occur only in evaluated conditions or executed statements and preserve earlier effects. A changed outer variable is read again by the next condition.

The body has a lexical scope. Existing mutable bindings can be updated; new local bindings cannot escape the body. Local mutable declarations execute their initializers on each iteration. Constants are evaluated during checking. Each declaration uses one static binding slot regardless of iteration count. Nested loops, conditionals and matches share the block-depth limit. Loop control uses no per-iteration stack growth; operations in its body can allocate and release dynamic objects. There is no iteration limit or automatic termination guarantee; `while true {}` runs until externally stopped. `break`, `continue`, and `for` are not implemented.

## Functions

```text
const digit_min = 48
const digit_max = 57

fn is_digit(code: int64) -> bool {
    if code < digit_min {
        return false
    }
    return code <= digit_max
}

print(is_digit(47))
print(is_digit(53))
```

The output is `false` then `true`. Functions are declared only at file scope. A signature names zero to sixteen parameters and an explicit result type, each `int64`, `bool`, `bytes`, `buffer`, `result<bytes>`, `result<buffer>` or `status`. Type names are contextual; `bytes` and `buffer` are also reserved built-in names. Parameter lists allow newlines between entries and after commas; a parameter name, colon and complete type stay on one line. Trailing commas are rejected. The opening body brace may follow the result type after newlines/comments; a completed definition requires a newline or EOF. There is no special entry function: file-level statements execute in order, while definitions are skipped.

Calls use parentheses and comma-separated expressions. They can appear in initializers, assignments, conditions, output, return expressions and other call arguments. Arguments evaluate left to right, then parameter values are copied before the body runs. Calls must match the declared count and types exactly. Zero-argument calls use `name()`. Discarded call statements, function values, nested definitions, closures, overloads, default parameters and void results are unsupported. `print` is the reserved output statement using the same parenthesized argument shape; it cannot be used as an expression result.

Parameter bindings are immutable run-time values, not compile-time constants. Assigning to one gives A0305; referring to one in a `const` initializer gives A0306. Use `var local = parameter` for a mutable local copy. Calls are run-time operations even with literal arguments and cannot initialize a compile-time constant. Arithmetic traps and earlier effects retain their normal order.

Each body has its own local scope and cannot read file-level mutable values or another activation's locals. Earlier file-scope compile-time constants are readable and their values are embedded, without run-time capture. Constants declared after the function, or inside a file-level branch, are not visible. Parameters and locals cannot shadow visible earlier constants. A name belonging to an invisible file-level mutable value may be reused locally. Function names are reserved against binding declarations throughout the file; calls may refer to functions defined later.

`return expression` exits the function immediately and must match its result type. A return outside a function, an invalid signature, a mismatched argument count or a missing return path gives A0308. Return-value type mismatches give A0303. Every path that may reach the body end must return: an `if` or `match` guarantees a return only when both arms do, and a loop is conservatively considered able to exit, including `while true`. Unreachable statements and unused function bodies are still checked. Direct or indirect recursive call graphs, including unreachable recursive calls, give A0309.

Function signatures are validated before the main statement pass, followed by function bodies in definition order. This pass order can report a file-level semantic error before a body error appearing earlier in the source. All checking finishes before execution or output-file creation. No call crosses a foreign-function or exported-module boundary in this subset.

## Limits and diagnostics

Limits are 256 total static bindings including parameters, declarations and match arm bindings in all function bodies, 64 functions, 16 parameters or call arguments, 64 active if/while/match levels per body, 128 expression group/call/index/unary levels, and 128 simultaneously live postfix operands. A function body does not consume a control-level slot; signature parentheses are not expression nesting. Expression parentheses count, including immediately after `print`. Every unary operator adds one level, including a minus directly before a literal. Flat left-associated binary chains add no group depth. The 257th binding/129th live operand gives A0307; excessive expression nesting gives A0202. Function/parameter/argument bounds are checked before bounded storage is accessed.

The generated ABI has a cumulative call-stack model limited to 64 KiB, including main, nested callees, return addresses and helpers. The dynamic runtime may add two helper frames of 168 bytes plus return addresses, a 352-byte helper allowance. Each generated frame is below 4096 bytes and is touched on entry. The acyclic graph is checked even for unused functions. This is a conservative generated-code resource policy, not a measurement of the C interpreter's exact stack use or operating-system API internals.

Order: complete UTF-8 validation, all CR validation, lexing, complete parsing, then signature/main/body checking as described above, return-path and call-graph/resource validation. Within a body, each destination/output name and declaration limit is checked before its postfix expression, then the final type. An if or while condition is checked before its blocks; a non-boolean condition gives A0303 at its root-expression token. Constant arithmetic uses the same semantic pass. A static error prevents execution and executable creation.

A0201 identifies a token violating the grammar. A0202 identifies the excessive group/unary sign or opening brace. The 65th active block gives A0202 on its opening brace; groups/unary operators retain their separate 128-level bound. Name/type checks identify the relevant name, number, or operator; an invalid final declaration type identifies the destination name. For a negative literal A0304 spans only the digits. Static errors include file/line/column and a stable diagnostic code, a message, a bounded source excerpt with carets, and a contextual hint where applicable. Invalid UTF-8 reports a byte offset. Runtime diagnostics retain the exact format below.

For an unknown variable or function the message quotes its name. A suggested spelling is shown only for a unique close candidate of the appropriate kind visible at that location; short names, ambiguous matches, out-of-scope names and distant guesses do not receive suggestions. No diagnostic silently declares a name. Source controls and non-ASCII excerpt scalars are escaped for unambiguous caret alignment and terminal safety; this changes only the display, not source interpretation.

Exit codes: 0 success, 1 static source error, 2 tool/I/O error, 3 runtime arithmetic or bounds error. Runtime stderr is exactly `A0401: int64 overflow.`, `A0402: division by zero.` or `A0403: byte index out of bounds.`, followed by LF, without path/position. A failed diagnostic write may leave a prefix but preserves code 3. A failed stdout write may leave partial output and stops execution with code 2.

## Implementation boundary

The current compiler and interpreter use C11, with Altai's own AMD64 generator for Windows x86-64. Modules, floating point, arrays, string variables, recursion and general-purpose memory allocation are absent. Square brackets support checked read-only byte/buffer indexing; append-only mutation is available through `append_byte`. The [native contract](native.md) describes targets, layout, and file handling. The lexer and compiler are not self-hosted; character-classification helpers written in Altai do not constitute a complete Altai lexer.

[English contents](../README.md#english) · [Русская версия](../ru/specification.md)
