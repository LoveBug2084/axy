**Please be aware this project is under construction and this is a very early draft**

# AXY Language Specification

AXY is a small medium-level language that compiles to BeebAsm 6502 assembly.
The name comes from the 6502 registers A, X and Y.

## Usage

```
./axy [-v] [-init] input.axy [output.asm]
```

- `input.axy` is the source file.
- `output.asm` is optional. If given, the generated assembly is written to
  that file. If omitted, it is written to stdout (the symbol report goes to
  stderr, keeping stdout a pure assembly stream).
- `-v` (anywhere in the command line) prints the symbol report after compiling:
  the current file's own variables (with their addresses), constants, and
  labels. Imported symbols are omitted. With no source file, `-v` lists the
  entire `.axysymbols` table instead — variables, constants and labels, each
  entry naming the file that declared it. Without it the compiler is silent on
  success.
- `-init` (anywhere in the command line) removes the `.axysymbols` state file
  before compiling, so a build chain starts from a clean symbol table. Used
  alone (`axy -init`) it only clears the file and exits, printing nothing;
  `axy -init -v` clears the file and then shows the symbol report. Start a fresh
  build with `axy -init -v library.axy`.
- The compiler refuses to run if the input and output are the same file.

All keywords are case-insensitive.

### `.axysymbols` — symbols across files

After a successful compile the compiler writes `.axysymbols` to the current
directory: every variable (with its allocated address), constant, and user
label. On the next run it reads that file back, so symbols declared in one
`.axy` source are known to compiles of the other files in the same directory:
a variable can be assigned, compared, and incremented from another file, and a
constant or label referenced by name.

Imported symbols are referenced but never re-emitted: the file that declares
them owns the BeebAsm definition, so files are intended to be assembled
together by BeebAsm. Each symbol may only be redeclared by the file that
originally declared it (its recorded origin) — recompiling that file is
harmless, while any other file redeclaring the name reports an error, whether
or not the definition matches. Each entry records the source file that
declared it (e.g. `var score = &70 ; lib.axy`), and duplicate-definition
errors report where the symbol was already defined:
`'score' already defined at &70 (lib.axy)`. Variables declared without an
`axyvar` continue after the highest address imported from `.axysymbols`.

There is no default variable base: declaring a `var` when no base has been
set — no `axyvar` in the source and no imported variables — reports an error.
An explicit `axyvar <address>` in the source always overrides the imported
base and re-anchors the chain. `.axysymbols` is not committed to version
control; delete it (or run `axy -init`) to reset the project's symbols.

## Generated output

The compiler produces a BeebAsm source file:

- header comment
- the source code, with each `var` and constant emitted as a BeebAsm symbol
  (`lives = &70`, `max = 5`, `greeting = "Hello"`) **at the point it is
  declared**, including ones never used in the code
- `org <address>` emitted where it appears in the source (see below)
- the generated code

Blank lines, comments, and the indentation of the source are preserved in the
output: each source line keeps its place and its leading whitespace in the
generated BeebAsm file.

Constants and variables therefore have to be declared before they are used:
BeebAsm resolves each symbol at the point it is defined, so a use before the
declaration line is reported by BeebAsm as an undefined symbol.

No entry or exit labels are generated automatically: labels such as `.start`
or `.end` are chosen by the user in the source. The only labels generated
automatically are for the internal `if`/`endif` `while`/`endwhile` and
`repeat`/`until` structures; they are defined inside BeebAsm `{ }` blocks, so they are
block-local and never conflict when several AXY programs are assembled
together.

Output is written as compilation progresses:

- **Success:** a complete `.asm` file.
- **Statement error:** a partial `.asm` containing everything generated up
  to the failing line.
- **Declaration error or missing input file:** no `.asm` is created.

## Comments

Anything after a `;` is a comment. Full-line comments are copied straight through to the generated output. A comment after AXY code is attached to the first assembler instruction that statement generates.

```
; this is a comment          -> ; this is a comment
a = 42          ; marker     -> lda #42 ; marker
if lives == 0 ; is the player dead ?  -> lda lives ; is the player dead ?
```

## Numbers

Number literals can be written in three bases:

- decimal, no prefix: `42`
- hexadecimal, `&` prefix: `&2A`
- binary, `%` prefix: `%101`

```
a = 42          ; lda #42
a = &2A         ; lda #&2A
a = %101        ; lda #%101
```

The notation you write is preserved in the generated assembly for number
literals in assignments and `if`/`while` comparisons. A malformed literal
(e.g. `&GG` or `%2`) reports `Invalid number`.

## Program start (`org`)

```
org <address>
```

Sets the program start address in the generated assembly. `<address>` is
passed through to BeebAsm as-is, so it may be a number in any supported base
(decimal, `&` hex, or `%` binary), a number constant, or any other BeebAsm
symbol:

```
org &2000
org 1234
org %1111000000000000
org base                ; constant base = &1900
```

If `<address>` names a constant, the constant is emitted as a symbol at its
declaration, so a chain like `org_addr = base` with `base = &1900` emits
`org org_addr` after both symbols and BeebAsm resolves it. `org` is emitted
where it appears in the source; code before it stays at BeebAsm's default
start address.

If no `org` is given, no `org` line is generated.

## Variable start (`axyvar`)

```
axyvar <address>
```

Sets the address of the next variable declared with `var`. It can appear
anywhere in the file, at any time: variables declared before it keep their
addresses, and variables declared after it continue from the new base. If no
`axyvar` is given, the `.axysymbols` file (see Usage) sets the base; with neither,
declaring a `var` is an error.

`<address>` may be a number in any supported base (decimal, `&` hex, or `%`
binary), or the name of a constant declared earlier in the file (chains are
resolved too):

```
axyvar &70
axyvar 1234
axyvar base                ; const base = &40
```

`axyvar` generates no assembly of its own; it only controls the addresses the
compiler assigns to `var`. The compiler does not range-check the value —
BeebAsm reports any address that is out of range for the instruction.

## Declarations

### var

```
var name
```

Declares a variable. Variables are allocated sequentially, starting at the
address set by the `axyvar` directive or the `.axysymbols` file (see Usage);
there is no default base, so declaring a `var` before either has set one is
an error. The registers `a`, `x`, `y` cannot be used as variable
names, and a name can only be declared once.

### constants

```
name = value
```

Declares a compile-time constant.

 Any line `name = value` where `name` is not already declared (variable, register, or
constant) and is not a keyword also declares a constant. A `var` must be
declared before its first assignment if it is to stay a variable. Constants
are immutable: a later `name = number` or `name += 1` reports an error, and
a name can only be declared once.

A constant is emitted once as a BeebAsm symbol at the point it is declared in
the generated output, keeping the notation you wrote in (`max = 5`,
`base = &1900`, `greeting = "Hello and welcome"`). Uses reference the constant
by name: `if score == max` compiles to `cmp #max`, and `a = max` compiles to
`lda #max`. All constants are emitted, including unused ones.

A constant may hold another constant's name (`maximum = limit`); the chain
resolves inside BeebAsm, so constants can even be shared across files that
are assembled together.

## Registers

The 6502 registers `a`, `x`, `y` are reserved and cannot be redeclared.
They can be assigned to, compared, and used with `+= 1` / `-= 1`.

## Assignment

```
dest = src
```

Copies `src` into `dest`. `dest` may be a register or variable; `src` may be
a register, variable, number literal, or constant. The correct addressing mode is
chosen matching the src type.

## Increment / Decrement

```
name += 1
name -= 1
```

Only `+= 1` and `-= 1` are supported. For `x`/`y` this compiles to
`inx`/`dex`/`iny`/`dey`; for variables it compiles to `inc name`/`dec name`.
`a` is not supported.

## Stack (`pha` / `pla`)

```
pha
pla
```

`pha` pushes the accumulator onto the hardware stack; `pla` pulls the top of
the stack into the accumulator. Neither takes arguments. They are reserved
keywords, so they cannot be used as variable or constant names.

## Labels and jumps

A label is defined by a line consisting of a dot followed by the label name.
The line is passed through to the assembly output as-is.

```
.loop
```

References use `jmp` with the bare name (no dot).

```
jmp loop
```

compiles to

```
jmp loop
```

Subroutine calls use `jsr` with the bare name; `rts` returns from a
subroutine.

```
jsr subroutine
rts
```

compile to

```
jsr subroutine
rts
```

The dot is only used when defining a label; references never use it. The
compiler does not check that a referenced label exists — it is passed
through and resolved by the assembler.

## If statement

```
if left op value
    ...
endif
```

Operators: `==` (equal) and `!=` (not equal).

- `left` may be a register (`a`, `x`, `y`) or a variable.
- `value` may be a number literal, a constant, or a variable.

Comparison compilation:

| Source               | Emitted                      |
| -------------------- | ---------------------------- |
| `if a == var`        | `cmp var`                    |
| `if a == 5`          | `cmp #5`                     |
| `if x == var`        | `cpx var`                    |
| `if x == const`      | `cpx #constname`             |
| `if y == var`        | `cpy var`                    |
| `if y == 5`          | `cpy #5`                     |
| `if var == 10`       | `lda var` / `cmp #10`        |
| `if var == othervar` | `lda var` / `cmp othervar`   |
| `if var == const`    | `lda var` / `cmp #constname` |

Rules:

- Register on the left: compare that register directly (`cmp`/`cpx`/`cpy`).
- Variable on the left: load it into A first (`lda var`), then `cmp`.
- Right side: a number literal compiles to an immediate (`#value`), a constant
  is referenced by name (`#constname`), and a variable stays as a bare memory
  operand.

`==` compiles to a `bne` to the end of the block (skip the body when not
equal); `!=` compiles to a `beq`.

The body of an `if`, `while`, and `repeat` is wrapped in BeebAsm `{ }` braces,
and the internal `.endifN`/`.endwhileN`/`.enduntilN` labels are block-local, so
several AXY programs can be assembled together without label conflicts.
Branches reference these labels with the bare name (`bne endif1`) — a dot is
only prefixed when a label is first defined.

## While loop

```
while left op value
    ...
endwhile
```

Uses the same operators and operand rules as `if`.

## Repeat loop

```
repeat
    ...
until left op value
```

Like a `while`, but the condition is tested **after** the body, so the body
always runs at least once. `until` loops back while the condition is **false**
and falls through when it is **true** (the opposite of `while`), which is why
`until left == value` compiles to a `bne` back to the top of the block and
`until left != value` to a `beq`.

Uses the same operators and operand rules as `if` and `while`.

## Assembler mode (`asm` / `endasm`)

```
asm
    <raw assembler lines>
endasm
```

`asm` switches to raw assembler mode: every line after it is copied to the
generated output verbatim — indentation, comments, and blank lines included —
with no AXY interpretation. `endasm` switches back to normal AXY compilation.
The `asm`/`endasm` lines themselves are not emitted.

Assembler mode is for code that AXY cannot express. Lines inside it are
completely ignored by the compiler: they are never parsed as declarations, so
a line like `foo = &1234` inside an asm block is just passed through and does
not declare an AXY constant.

Symbols are shared through the generated file. Constants and variables must
be declared in AXY code to be usable inside an asm block (AXY symbols are
emitted in the header, before the `org`), and a constant defined inside an
asm block can be referenced from later AXY code by name, where it compiles to
an immediate (`#name`). A variable defined inside an asm block is not
knowable to AXY as a variable — AXY would treat its name as a constant.

## Errors

Errors print the offending source line with a `^` marker under the position,
followed by `Line N: message`:

```
temp += 3
        ^
Line 2: Only +=1 and -=1 supported
```

## Example

```
var count
const target = 5

count = 0
.loop
    if count == target
        jmp done
    endif
    count += 1
    jmp loop
.done
    a = count
```

## Status

| Feature              | Status      |
| -------------------- | ----------- |
| `var`                | implemented |
| constants            | implemented |
| assignment           | implemented |
| `+= 1` / `-= 1`      | implemented |
| labels (`.name`)     | implemented |
| `jmp`                | implemented |
| `jsr` / `rts`        | implemented |
| `if ... endif`       | implemented |
| `while ... endwhile` | implemented |
| `repeat ... until`   | implemented |
| `pha` / `pla`        | implemented |
| `org` / `axyvar`     | implemented |
| `asm` / `endasm`     | implemented |
| comments             | implemented |
| `.axysymbols` symbol file | implemented |

---

# Change Log

## 2026-08-11 — `.axysymbols` state file

- The old `.axyvar` file (a single number) is replaced by `.axysymbols`, which
  stores the full symbol table: every variable with its allocated address,
  every constant, and every user label. A successful compile writes the file;
  the next compile in the same directory reads it back, so a variable can be
  assigned, compared and incremented from another `.axy` file, and constants
  or labels referenced by name.
- Symbols are shared but only defined once: an imported symbol is referenced
  without being re-emitted, so the `.axy` files are intended to be assembled
  together by BeebAsm. A symbol may only be redeclared by the file that
  originally declared it — recompiling that file is harmless, while any other
  file redeclaring the name reports an error, whether or not the definition
  matches.
- New `-init` flag removes `.axysymbols` so a build starts from a clean symbol
  table. Used alone it is silent; `-init -v` removes the file then prints the
  standard symbol report.
- The `-v` symbol report now lists only the current file's own symbols and
  adds a `Labels` section. With no source file, `-v` lists the entire
  `.axysymbols` table (variables, constants, labels), each entry naming the
  file that declared it.
- Variables declared without an `axyvar` continue after the highest address
  imported from `.axysymbols`.
- Each `.axysymbols` entry records the source file that declared it, and
  duplicate-definition errors name that file, e.g.
  `'score' already defined at &70 (lib.axy)`.

## 2026-08-11 — case-insensitive `endasm`; two-pass refactor

- `endasm` is now recognised case-insensitively (like every other keyword) and
  may carry a trailing comment, so `ENDASM ; done` inside an `asm` block ends
  it. Previously the block-exit check was case-sensitive and required the line
  to be exactly `endasm`.
- Internal: `translate()` split into `_pass1` (declaration pass) and `_pass2`
  (statement pass), which now share the token-based asm/`endasm` handling.

## 2026-08-10 — repeat / until loop

- `repeat` ... `until left op value` adds a do-until loop: the body always runs
  at least once, `until` tests the condition after the body, loops back while it
  is false, and falls through when it is true. Compiles to a `.repeatN` /
  `.enduntilN` block with a `bne`/`beq` back to the top.

## 2026-08-10 — blank lines preserved in generated output

- Blank and whitespace-only lines in AXY source are now copied through to the
  generated output instead of being stripped, alongside comments and
  indentation.

## 2026-08-10 — declarations emitted at their definition site

- `var` and constant declarations are no longer collected into a header block:
  each is emitted as a BeebAsm symbol at the point it is declared in the
  source. Uses before the declaration line are left to BeebAsm to report.
- `org` is now emitted where it appears in the source instead of at the top.

## 2026-08-10 — comments in generated output

- Full-line comments in AXY source are copied straight through to the generated
  output as their own comment lines.
- A comment after AXY code is appended to the first assembler instruction that
  statement generates (labels and block braces are skipped).

## 2026-08-10 — assembler mode (`asm` / `endasm`)

- `asm` switches to raw assembler mode: every line after it is copied to the
  generated output verbatim, with no AXY interpretation, until `endasm`
  switches back.
- Lines inside an asm block are ignored by the compiler entirely — they are
  never parsed as declarations, so they cannot accidentally declare AXY
  constants.
- Symbols are shared through the generated file: AXY constants and variables
  are usable inside an asm block, and a constant defined inside an asm block
  can be referenced from AXY code as an immediate (`#name`).

## 2026-08-10 — `.axyvar` state file

- On startup the compiler reads `.axyvar` from the current directory; if it
  holds a valid number it initialises the variable address counter, so a
  source that declares `var` without an `axyvar` of its own continues from
  where the previous file ended.
- After a successful compile the current counter value is written back to
  `.axyvar`. Failed compiles leave it untouched, and an explicit `axyvar` in
  the source overrides the file's value and re-anchors the chain.

## 2026-08-10 — constants as BeebAsm symbols

- Constants are emitted once as BeebAsm symbols in the generated output,
  before the `org` line, keeping the notation they were written in
  (`max = 5`, `base = &1900`, `greeting = "Hello"`).
- Uses reference a constant by name: `a = max` emits `lda #max`, and
  `if score == max` emits `cmp #max`. Constant chains (`maximum = limit`)
  resolve in BeebAsm, and constants may be shared across files assembled
  together.
- The `const` keyword is optional: any `name = value` line where the name is
  not already declared and is not a keyword declares a constant.
- Constants are immutable: reassignment or `+=`/`-=` on one reports an error.

## 2026-08-10 — `org` and `axyvar` directives

- `org <address>` sets the program start address. The argument is passed
  through verbatim, so it may be a number in any base or any BeebAsm symbol.
  If no `org` is given, none is generated.
- `axyvar <address>` sets the address of the next `var` declaration. It may
  appear anywhere in the file; a mid-file `axyvar` resets the base for
  subsequent variables only. The argument may be a number or the name of a
  constant declared earlier.
- `var` has no default base address: allocating one requires an `axyvar` in
  the source or a `.axyvar` file. The compiler performs no range checking on
  variable addresses — BeebAsm reports any that are out of range.

## 2026-08-10 — labels

- A label is defined with a dot (`.loop`) but every reference to it is bare
  (`jmp loop`, `bne endif1`); the dot is only ever prefixed when the label is
  first defined.
- The compiler generates no automatic labels; the source supplies its own, so
  several AXY programs can be assembled together by BeebAsm.
- The internal labels generated for `if`/`while` are wrapped in BeebAsm
  `{ }` braces, making them block-local so they never collide across programs.

## 2026-08-09 — initial compiler

- Translates `.axy` files to BeebAsm 6502 assembly: declarations (`var`,
  `const`), assignment (`a = 5`, `score = value`), `+= 1` / `-= 1`, `jmp`,
  `jsr` / `rts`, `if ... endif`, and `while ... endwhile`.
- Number literals accept decimal, `&` hex and `%` binary; the notation is
  preserved in the generated output.
- Two-pass streaming compilation: a declarations pass then a statements pass,
  each reading the source one line at a time. Errors show the offending line
  with a `^` marker.
- Output is written to the given file, or to stdout when none is given;
  writing over the input file is rejected.
- The generated output header carries a timestamp.
