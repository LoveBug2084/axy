# 16-bit Variable Implementation Plan for AXY Compiler

## Overview

Add `var16` keyword support for 16-bit variables in the AXY compiler, enabling 2-byte variable handling with full carry-flag-aware arithmetic.

## Key Changes

### 1. `process_declaration()` - Add `var16` keyword handling

- Detect `var16` keyword (case-insensitive)
- Allocate 2 consecutive addresses for the variable
- Increment `next_var` by 2 after declaration
- Validate name not in ('a', 'x', 'y') - those are reserved registers
- Check for duplicate declarations (existing behavior preserved)

### 2. `load_symbols()` - Read `axyvar` directly

- Parse `axyvar` entry from `.axysymbols` file
- Set `self.next_var` directly from the stored address
- If no `axyvar` entry exists and no `axyvar` in source: `next_var` stays `None`
- Error on first `var` or `var16` declaration if no base address set
- Fallback to max address calculation only if needed (preserve existing behavior)

### 3. `save_symbols()` - Write `axyvar` entry

- Write `axyvar &{next_var:04X}` as the first line of `.axysymbols`
- Preserve variable base address across compiles
- Enables clean symbol import on subsequent compiles

### 4. 16-bit Math Operations

- `process_incdec`: 16-bit increment/decrement with full carry management
  - `var16 += 1`: `clc / lda var_low / adc #1 / sta var_low / lda var_high / adc #0 / sta var_high`
  - `var16 -= 1`: `sec / lda var_low / sbc #1 / sta var_low / lda var_high / sbc #0 / sta var_high`
  - `var16 += var`: `lda var1_low / clc / adc var_low / sta var_low / lda var_high / adc var_high / sta var_high`
  - `var16 -= var`: similar with `sbc` instead of `adc`
- `process_assignment`: 16-bit variable assignment (move both bytes)
- `_gen_assign_immediate`: 16-bit immediate load (`lda var_low / adc #0 / sta var_high`)
- 16-bit comparison in `process_if`/`process_while`/`process_until` (compare low bytes, optional high byte)

### 5. Symbol Report Updates

- Show `var16` entries separately or with byte size indicator
- Format: `var16 name = &addr ; 2 bytes`
- Maintain backward compatibility with `var` entries

### 6. `.axysymbols` Format

- Existing `var` entries: `var name = &addr ; filename`
- New `var16` entries: `var16 name = &addr ; filename` (4 hex digits for address)
- `axyvar` entry at file top: `axyvar &XXXX ; filename`

### 7. Backward Compatibility

- Existing `var` variables continue to work unchanged
- New `var16` files can coexist with existing `var` files in same directory
- `.axysymbols` format maintains compatibility with older compile versions
- `axyvar` entry optional on first compile (errors if `var` declared without base)

### 8. User-facing Behavior

- `var16 myvar` declares 2-byte variable at consecutive addresses
- `myvar += 1` / `myvar -= 1` performs full 16-bit arithmetic with carry
- `myvar += constant` / `myvar -= constant` with constant value
- `myvar += another_var` / `myvar -= another_var` with another variable
- `axyvar &addr` in source or `.axysymbols` sets base for both `var` and `var16`
- Compiler errors if no base address set and `var`/`var16` declared

## Design Rationale

- `var16` explicit keyword over `varw`/`vbyte` for clarity and self-documentation
- `var` remains 8-bit, `var16` is 16-bit - clear distinction
- `axyvar` in `.axysymbols` makes symbol import deterministic across compiles
- Full carry-flag-aware 16-bit arithmetic matches assembly best practices
- Minimal disruption to existing code and workflow

## Implementation Priority

1. Add `var16` keyword handling and 2-byte address allocation
2. Implement `axyvar` read/write in `load_symbols()`/`save_symbols()`
3. Add 16-bit math operations in `process_incdec` and `_gen_*` functions
4. Update symbol report format
5. Test with existing test suite and verify backward compatibility