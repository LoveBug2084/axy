###### Please be aware this project is under construction and this is a very early draft of the language, much is missing !

# AXY Language Reference

AXY is a small, medium-level language that compiles to [BeebAsm](https://github.com/dominicbeesley/BeebAsm)
6502 assembly for the BBC Micro and Electron. The name comes from the 6502
registers **A**, **X** and **Y**.

AXY is deliberately tiny. It gives you a handful of readable statements
(`if`, `while`, `repeat`, `+= 1`, …) that expand to idiomatic 6502 code, and
lets you drop into raw assembly whenever you need something it cannot express.
Everything compiles to a single `.asm` file that BeebAsm assembles.

All keywords are **case-insensitive** (`If`, `ENDWHILE`, `PhA` all work).

---

## Contents

1. [Command line](#command-line)
2. [Program structure](#program-structure)
3. [Comments](#comments)
4. [Numbers](#numbers)
5. [Declarations](#declarations)
6. [Registers](#registers)
7. [Directives](#directives)
8. [Statements](#statements)
9. [Operators](#operators)
10. [Flow control](#flow-control)
11. [Raw assembler mode](#raw-assembler-mode)
12. [Multiple source files](#multiple-source-files)
13. [Symbols across files](#symbols-across-files)
14. [Errors](#errors)
15. [What AXY does not have](#what-axy-does-not-have)
16. [Complete example](#complete-example)

---

## Command line

```
./axy [-v] [-init] input.axy [output.asm]
```

- `input.axy` is the source file.
- `output.asm` is optional. If given, the generated assembly is written to that
  file. If omitted it is written to stdout (the symbol report goes to stderr,
  keeping stdout a pure assembly stream).
- `-v` prints a symbol report after compiling: variables, constants and labels,
  grouped by the file that declared them under `file.axy:` headings. With no
  source file, `-v` prints the whole `.axysymbols` table instead.
- `-init` deletes the `.axysymbols` state file before compiling, so a build
  starts from a clean symbol table. Used alone it just clears the file.
- Compiling with the same path as input and output is refused.

---

## Program structure

A `.axy` file is a sequence of lines. Each line is either a **directive**, a
**declaration**, a **statement**, a label, a blank line, or a comment. Blank
lines, comments, and indentation are preserved in the generated `.asm`.

The compiler runs two passes:

1. **Declaration pass** — builds the symbol table (`var`, constants, labels).
2. **Statement pass** — streams the assembly out line by line.

Declarations are emitted as BeebAsm symbols **at the point they appear in the
source**, so a name must be declared before it is used — BeebAsm resolves each
symbol where it is defined.

```
var count    ; emitted as: count = &70
target = 5   ; emitted as: target = 5
```

There are no automatic entry or exit labels; the source supplies its own
(`.start`, `.done`, …). Labels generated internally for `if`/`while`/`repeat`
blocks live inside BeebAsm `{ }` blocks, so they are block-local and never
conflict.

---

## Comments

Anything after a `;` is a comment.

- A full-line comment is copied straight through to the generated output.
- A comment after code is attached to the first assembler instruction that
  statement generates.

```
; this is a comment          ; ->  ; this is a comment
a = 42          ; marker     ; ->  lda #42 ; marker
if lives == 0   ; dead ?     ; ->  lda lives ; dead ?
```

---

## Numbers

Three number bases, written exactly as BeebAsm reads them:

| Base        | Prefix | Example |
| ----------- | ------ | ------- |
| Decimal     | —      | `42`    |
| Hexadecimal | `&`    | `&2A`   |
| Binary      | `%`    | `%101`  |

```
a = 42       ; lda #42
a = &2A      ; lda #&2A
a = %101     ; lda #%101
```

The notation you write is preserved in the generated assembly. A malformed
literal (`&GG`, `%2`) reports `Invalid number`.

---

## Declarations

### `var name`

Declares a variable. Variables are allocated sequentially, one byte each,
starting at the address set by `axyvar` or imported from `.axysymbols`. There
is **no default base**: declaring a `var` before either has set one is an
error.

```
var lives
var score
var name
```

Rules:

- The registers `a`, `x`, `y` cannot be used as variable names.
- A name can only be declared once (per its owning file).
- A variable already recorded in `.axysymbols` keeps its allocated address, so
  recompiling its file never shifts it.

### Constants

A constant is declared by a bare assignment whose name is not already declared
and is not a keyword:

```
max = 5
base = &1900
greeting = "Hello and welcome"
screen = %10100000
```

Rules:

- A name used on the left of `=` first becomes a constant; once a variable is
  declared with that name, `name = value` assigns to it instead.
- Constants are **immutable**: reassigning one, or applying `+= 1` / `-= 1`,
  is an error.
- A constant is emitted once as a BeebAsm symbol at its declaration, keeping
  the notation it was written in. All constants are emitted, including unused
  ones.
- Uses reference the constant by name: `a = max` compiles to `lda #max` and
  `if score == max` to `cmp #max`.
- A constant may hold another constant's name (`maximum = limit`); the chain
  resolves inside BeebAsm.

---

## Registers

The 6502 registers `a`, `x`, `y` are reserved and cannot be redeclared. They
can be assigned to, compared, and used with `+= 1` / `-= 1`.

---

## Directives

### `org <address>`

Sets the program start address. The address is passed through to BeebAsm
as-is, so it may be a number in any base, a constant, or any BeebAsm symbol:

```
org &2000
org base
```

`org` is emitted where it appears in the source; code before it stays at
BeebAsm's default start address. If no `org` is given, none is generated.

### `axyvar <address>`

Sets the address of the **next** variable declared with `var`. It can appear
anywhere in the file:

- variables declared before it keep their addresses;
- variables declared after it continue from the new base.

`<address>` may be a number in any base or the name of a constant declared
earlier (chains resolve too). An explicit `axyvar` overrides any base imported
from `.axysymbols`. `axyvar` generates no assembly of its own, and the
compiler does not range-check the address.

```
axyvar &70
axyvar base
```

### `include <file>`

Splices another `.axy` file into the current one — declarations join the same
symbol table and code is emitted inline, exactly as if the file's contents had
been written at the `include` line. See [Multiple source files](#multiple-source-files).

---

## Statements

### Assignment — `dest = src`

```
dest = src
```

Copies `src` into `dest`.

- `dest` may be a register (`a`, `x`, `y`) or a variable.
- `src` may be a register, a variable, a number literal, or a constant.
- The correct addressing mode is chosen from the `src` type: a number or
  constant loads an immediate (`lda #5`), a variable or register is copied
  directly.

```
a = 5           ; lda #5
a = b           ; lda b
x = a           ; tax
score = 0       ; lda #0 / sta score
score = a       ; sta score
```

### Increment / decrement — `name += 1`, `name -= 1`

Only `+= 1` and `-= 1` are supported.

| Target       | `+= 1`        | `-= 1`        |
| ------------ | ------------- | ------------- |
| Register `x` | `inx`         | `dex`         |
| Register `y` | `iny`         | `dey`         |
| Variable     | `inc name`    | `dec name`    |
| Register `a` | not supported | not supported |

```
count += 1     ; inc count
x -= 1         ; dex
```

### Stack — `pha`, `pla`

`pha` pushes the accumulator onto the hardware stack; `pla` pulls the top of
the stack back into the accumulator. Neither takes arguments. They are
reserved keywords and cannot be used as names.

```
pha
a = 7
pla
```

### Labels — `.name`

A label is defined by a line consisting of a dot followed by the name. The
dot is only ever used when **defining** a label; every reference is bare:

```
.loop
    ...
    jmp loop
```

The compiler does not check that a referenced label exists — it is passed
through and resolved by the assembler.

### Jumps — `jmp`, `jsr`, `rts`

```
jmp label       ; unconditional jump
jsr subroutine  ; call a subroutine
rts             ; return from a subroutine
```

All three pass through to assembly as-is (bare label, no dot).

---

## Operators

AXY has a deliberately small operator set:

| Operator | Meaning                           | Used in                |
| -------- | --------------------------------- | ---------------------- |
| `=`      | assignment / constant declaration | assignments, constants |
| `+=`     | increment by one or more         | variables, `x`, `y`    |
| `-=`     | decrement by one or more         | variables, `x`, `y`    |
| `==`     | equal                             | `if`, `while`, `until` |
| `!=`     | not equal                         | `if`, `while`, `until` |

There are no arithmetic expressions — see
[What AXY does not have](#what-axy-does-not-have).

### Extended `+=`/`-=` operators (since 2026-08-12)

`+=` and `-=` now support any numeric value, variables, and constants on the
right-hand side:

| Syntax | Generated 6502 | Notes |
|--------|----------------|-------|
| `name += 1` / `name -= 1` | `inc name` / `dec name` (variable)<br>`inx` / `dex` / `iny` / `dey` (x/y) | Optimized path - preserved from before |
| `name += value` / `name -= value` (number literal) | `lda name / clc / adc #value / sta name` (variable)<br>`clc / adc #value` / `sec / sbc #value` (register `a`) | `clc` always precedes `adc`; `sec` always precedes `sbc` |
| `name += var` / `name -= var` (variable) | `lda name / clc / adc var / sta name` (variable)<br>`lda name / sec / sbc var / sta name` (subtraction) | Uses absolute addressing for the RHS variable |
| `name += const` / `name -= const` (constant) | `lda name / clc / adc #const / sta name` (variable)<br>`lda name / sec / sbc #const / sta name` (subtraction) | Constant chain resolves in BeebAsm (e.g. `maximum = limit`) |

- `x`/`y` registers transfer through accumulator: `txa`/`tya` → math → `tax`/`tay`
- Negative numbers not supported as a single token (use `var -= 5` not `var += -5`)
- Carry flag must be managed by user in complex sequences

---

---

## Flow control

`if`, `while` and `until` share the same operands:

- `left` may be a register (`a`, `x`, `y`) or a variable.
- `value` may be a number literal, a constant, or a variable.
- Operators are `==` and `!=` only.

How a comparison compiles:

| Condition            | Emitted                      |
| -------------------- | ---------------------------- |
| `if a == var`        | `cmp var`                    |
| `if a == 5`          | `cmp #5`                     |
| `if x == var`        | `cpx var`                    |
| `if x == const`      | `cpx #constname`             |
| `if y == 5`          | `cpy #5`                     |
| `if var == 10`       | `lda var` / `cmp #10`        |
| `if var == othervar` | `lda var` / `cmp othervar`   |
| `if var == const`    | `lda var` / `cmp #constname` |

Rules:

- Register on the left: compare that register directly (`cmp`/`cpx`/`cpy`).
- Variable on the left: load it into `a` first (`lda var`), then `cmp`.
- Right side: a number compiles to an immediate (`#5`), a constant is
  referenced by name (`#constname`), a variable stays a bare memory operand.
- `==` compiles to a `bne` over the body (skip when not equal); `!=` compiles
  to a `beq`.

The body of every `if`, `while` and `repeat` is wrapped in BeebAsm `{ }`
braces, and the internal `.endifN` / `.endwhileN` / `.enduntilN` labels are
block-local.

### `if ... endif`

```
if lives == 0
    ...
endif
```

### `while ... endwhile`

```
while x != 10
    ...
endwhile
```

`while` tests the condition **before** the body, like a standard loop.

### `repeat ... until`

```
repeat
    ...
until x == 10
```

Like a `while`, but the condition is tested **after** the body, so the body
always runs at least once. `until` loops back while the condition is **false**
and falls through when it is **true** — the opposite of `while`, which is why
`until x == 10` compiles to a `bne` back to the top.

---

## Raw assembler mode

```
asm
    LDA #&20
    JSR &FFEE
endasm
```

`asm` switches to raw assembler mode: every line after it is copied to the
generated output verbatim — indentation, comments and blank lines included —
with no AXY interpretation. `endasm` switches back. The `asm`/`endasm` lines
themselves are not emitted.

Notes:

- Lines inside an asm block are never parsed as declarations, so a line like
  `foo = &1234` inside an asm block does **not** declare an AXY constant.
- A constant defined inside an asm block can be referenced from later AXY code
  by name, where it compiles to an immediate (`#name`). A variable defined
  there is not knowable to AXY as a variable.

---

## Multiple source files

### `include`

```
include math.axy
include draw.axy
include sound.axy
```

`include` splices another file into the compile:

- File names are relative to the current working directory.
- Everything is allowed inside an included file: `var`, constants, `org`,
  `axyvar`, labels, jumps, loops, `asm` blocks, and further `include`s.
- Symbols declared in an included file carry that file's name as their origin,
  so a variable allocated by `math.axy` is reported under a `math.axy:`
  heading and stored as `var foo = &70 ; math.axy` in `.axysymbols`.
- Including the same file twice is harmless: after the first include, later
  includes of that file are silently ignored. A diamond of includes
  (main → draw and sound, both → math) therefore compiles math exactly once.
- An `include` inside an `asm` block is passed through verbatim and not
  expanded.
- Include depth is capped at 50.

Errors: `Expected 'include <file>'`, `Included file not found: 'X'`,
`Circular include of 'X'`.

**Gotcha:** don't include a file *and* also assemble the separate `.asm` it
produces — its labels and constants would be emitted twice and BeebAsm would
reject the duplicates. Build the combined program from a `main.axy` that
includes everything, and assemble only `main.asm`.

---

## Symbols across files

After a successful compile, the compiler writes `.axysymbols` into the current
directory: every variable (with its allocated address), constant and label.
On the next run it reads that file back, so a variable declared in one `.axy`
file can be assigned, compared and incremented from another, and constants and
labels referenced by name.

Key rules:

- Imported symbols are referenced but **never re-emitted**: the file that
  declares them owns the BeebAsm definition. Files are intended to be
  assembled together by BeebAsm.
- Each symbol may only be redeclared by the file that originally declared it
  (its recorded origin). Recompiling that file is harmless; any other file
  redeclaring the name reports an error:
  `'score' already defined (lib.axy)`.
- A variable already present keeps its recorded address; variables declared
  without an `axyvar` continue after the highest imported address.
- An explicit `axyvar` in the source always overrides the imported base.
- `.axysymbols` is not committed to version control. Delete it or run
  `axy -init` to reset the project's symbols.

```
; example .axysymbols
var libvar = &70 ; lib.axy
const libconst = 42 ; lib.axy
label .subroutine ; lib.axy
```

---

## Errors

Errors print the offending source line with a `^` marker under the position,
followed by `file.axy: line N: message` — the filename because a single compile
can cover more than one file:

```
temp += 3
        ^
program.axy: line 2: Only +=1 and -=1 supported
```

---

## What AXY does not currently have

AXY is intentionally minimal but its a bit too minimal at the moment but this will change. Currently there are no:

- **Arithmetic expressions** — no `+`, `-`, `*`, `/` beyond the built-in
  `+= 1` / `-= 1`. Values are single names, numbers, or string constants.
- **Arithmetic or math functions** — no `abs()`, `sin()`, random numbers,
  etc. For now you will have to write the assembly in an `asm` block instead.
- **Arrays or structs** — a `var` is a single byte at a fixed address.
- **Expressions in conditions** — only `reg/var op value` with `==` / `!=`.
- **Loops with conditions other than `==` / `!=`** — no `<`, `>`, etc.
- 
- **String or data directives** — write them in an `asm` block.

---

## Complete example

```
org &2000
axyvar &70

; --- declarations ---
var lives
var score
max = 3

; --- program ---
lives = max
score = 0

.loop
    if score == max
        jmp done
    endif

    score += 1
    jmp loop

.done
    a = score
    rts
```

Compiling `axy example.axy` produces assembly in which `lives = &70`,
`score = &71`, `max = 3` and the `org` line appear at their source positions,
and every statement expands to idiomatic 6502 instructions ready for BeebAsm.
