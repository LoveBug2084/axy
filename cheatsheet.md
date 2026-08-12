AXY Quick Reference
===================

Directives
----------
- `var name` - declare variable
- `org <address>` - set program start address
- `axyvar <address>` - set next variable base address
- `include <file>` - include another .axy file
- `asm` / `endasm` - raw assembler mode

Statements
----------
- `dest = src` - assignment
- `name += 1` / `name -= 1` - increment/decrement
- `name += value` / `name -= value` - extended operators
- `name += var` / `name -= var` - variable RHS
- `name += const` / `name -= const` - constant RHS
- `a += value` / `a -= var` - register A operations
- `x += 1` / `y -= 1` - optimized x/y operations
- `pha` / `pla` - stack operations
- `jmp label` / `jsr sub` / `rts` - jumps and calls

Flow Control
------------
- `if left op value` / `endif` - conditional
- `while left op value` / `endwhile` - while loop
- `repeat` / `until left op value` - do-until loop

Labels
------
- `.name` - define label (reference as bare `name`)

Extended +=/-= Operators (since 2026-08-12)
-------------------------------------------

| Syntax | Generated 6502 | Notes |
|--------|----------------|-------|
| `name += 1` (variable) | `inc name` | Optimized |
| `name += 1` (`x`) | `inx` | via accumulator |
| `name += 1` (`y`) | `iny` | via accumulator |
| `name += 5` (variable) | `lda name / clc / adc #5 / sta name` | `clc` before `adc` |
| `name -= 5` (variable) | `lda name / sec / sbc #5 / sta name` | `sec` before `sbc` |
| `name += var` (variable) | `lda name / clc / adc var / sta name` | absolute addressing |
| `name += const` (constant) | `lda name / clc / adc #const / sta name` | chain resolves in BeebAsm |
| `a += 10` (register A) | `clc / adc #10` | carry must be managed |
| `a -= var` (register A) | `sec / sbc var` | |
| `x += var` (register X) | `txa / clc / adc var / tax` | via accumulator |
| `y -= 2` (register Y) | `tya / sec / sbc #2 / tay` | via accumulator |
| `x += 1` | `inx` | optimized |
| `y -= 1` | `dey` | optimized |

Comparison Operators (if/while/repeat)
--------------------------------------
- `==` (equal) compiles to `bne` over body
- `!=` (not equal) compiles to `beq` over body

Right side types:
- Number literal → immediate (`#5`)
- Constant → by name (`#constname`)
- Variable → bare operand (`var`)

Labels are block-local (`.endifN`, `.endwhileN`, `.enduntilN`)

Raw Assembler Mode
------------------
- `asm` / `endasm` - lines passed through verbatim
- Inside asm: AXY code is NOT parsed
- Constants defined in asm are usable from AXY code as `#name`
- Variables defined in asm are NOT knowable to AXY

Symbols Across Files
--------------------
- `.axysymbols` stores all variables, constants, and labels
- Imported symbols keep their declaring file's origin
- Recompile the owning file is harmless; other files redeclaring report error
- Run `axy -init` to reset `.axysymbols`

See `language.md` for full language reference.