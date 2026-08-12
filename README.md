# AXY Compiler

Translates `.axy` files to [BeebAsm](https://github.com/dominicbeesley/BeebAsm) 6502 assembly for the BBC Micro and Electron.

The name comes from the 6502 registers **A**, **X** and **Y**.

## Usage

```
./axy [-v] [-clean] input.axy [output.asm]
```

- `input.axy` is the source file
- `output.asm` is optional; if omitted, assembly is written to stdout
- `-v` prints the symbol report after compiling
- `-clean` removes the `.axysymbols` state file before compiling, use this option when performing a clean build

## Documentation

- **[Full language reference](language.md)** — comprehensive syntax, directives, flow control, symbols and extended `+=`/`-=` operators
- **[Quick cheatsheet](cheatsheet.md)** — common patterns and operator reference for quick lookup

## Changelog

## 2026-08-11 — `include` directive

- `include file.axy` splices another source file into the compile: its
  declarations join the shared symbol table and its code is emitted inline in
  the `.asm`. File names are relative to the working directory.
- Included symbols take the included file's name as their origin in
  `.axysymbols` and in the `-v` report.
- Including the same file twice is silently ignored after the first include.
  Circular includes, missing files, malformed directives and excessive depth
  (over 50) are errors. An `include` inside an `asm` block is passed through
  as raw text and not expanded.
- The compiler now tracks which source file each line comes from: symbol
  origins, `const_def_lines` and error messages are all per-file, so equal
  line numbers across files cannot collide.
- Error output now shows the source file: `file.axy: N: message` instead of
  `Line N: message`.
- Error output spells the line number out: `file.axy: line N: message`
  (e.g. `program.axy: line 2: Only +=1 and -=1 supported`), so the number is
  self-explanatory and cannot be mistaken for a column.

## 2026-08-11 — `-v` symbol report grouped by file

- With no source file, `-v` now groups the `.axysymbols` listing by type, then
  by the file that declared each symbol, then by name. Each file's entries sit
  under a `file.axy:` heading instead of carrying a `file:` prefix, so
  symbols from different files are no longer interleaved.
- The report shown after compiling a file uses the same grouping: its own
  symbols sit under a `file.axy:` heading and are sorted by name.

## 2026-08-11 — `-v` listing format; owner recompile fix

- With no source file, `-v` now lists each symbol as `filename: name`
  (e.g. `lib.axy: score = $70`) — the declaring file comes first, before the
  symbol — instead of appending it after.
- Fixed: recompiling the file that owns a variable no longer fails. A variable
  already present in the imported `.axysymbols` table keeps its recorded
  address — only new variables are allocated from the counter and advance it —
  so the state file stays stable and no addresses are skipped.

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

## 2026-08-12 — Extended `+=`/`-=` operators

- `+=` and `-=` now support any numeric value (not just `1`), variables, and
  constants on the right-hand side.
- `name += value` / `name -= value` where value is a number literal generates:
  `lda name / clc / adc #value / sta name` (for variables) or `clc / adc #value`
  (for register `a`).
- `name += var` / `name -= var` where value is a variable generates:
  `lda name / clc / adc var / sta name` (or `sec / sbc var` for subtraction).
- `name += const` / `name -= const` where value is a constant generates:
  `lda name / clc / adc #const / sta name` (BeebAsm resolves chains).
- `x`/`y` registers use `txa`/`tya` to transfer through accumulator.
- Optimization preserved: `+= 1`/`-= 1` on variables uses `inc`/`dec`, and on
  `x`/`y` uses `inx`/`dex`/`iny`/`dey`.
- Always preceeds `adc` with `clc` and `sbc` with `sec`.

## 2026-08-14 — Include directive line numbers

- Circular include errors now show the line number and `^` marker (e.g.
  `file.axy: line 100: Circular include of 'file.axy'`).
- `Include file not found` errors at the top level now show the line number.
- All include-related errors now format consistently with other errors in the
  compiler.
- The circular include check was moved from before the file processing loop to
  inside the loop where `line_num` is available.