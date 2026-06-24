# 6.3 — The Line-Number Program

When GDB shows you the source line for a crash, or a profiler attributes time to
`foo.cpp:42`, it's consulting DWARF's **line-number program** in `.debug_line`.
This is a beautiful piece of engineering: a tiny bytecode interpreter that
reconstructs a giant address→line table without storing it. This chapter
explains it.

---

## 6.3.1 The problem: a huge mapping, compactly

Every machine instruction maps to some source line. A naive table would have one
row per address — enormous. But the mapping is *mostly monotonic and
incremental*: consecutive instructions usually map to the same or the next line.
DWARF exploits this with a **state machine** driven by a bytecode program.

```
   What we want (the "line table" / "line matrix"):
   ┌────────────┬──────┬──────┬─────────┬───────────────────────────────┐
   │ Address    │ File │ Line │ Column  │ flags (is_stmt, end_seq, ...) │
   ├────────────┼──────┼──────┼─────────┼───────────────────────────────┤
   │ 0x1149     │ 1    │ 4    │ 0       │ is_stmt                       │
   │ 0x1151     │ 1    │ 5    │ 0       │ is_stmt                       │
   │ 0x1160     │ 1    │ 6    │ 0       │ is_stmt                       │
   │ 0x1170     │ 1    │ 6    │ 0       │ end_sequence                  │
   └────────────┴──────┴──────┴─────────┴───────────────────────────────┘

   We DON'T store this table. We store a PROGRAM that REGENERATES it.
```

---

## 6.3.2 The state machine

The line-number program drives a register set; executing each opcode updates
registers, and certain opcodes **append the current register state as a row** to
the table.

```
   State machine registers (key ones):
     address    : current machine address       (starts 0)
     file       : current source file index      (starts 1)
     line       : current source line            (starts 1)
     column     : current column                 (starts 0)
     is_stmt    : is this a recommended breakpoint location? (starts per-header)
     end_sequence : marks the end of a contiguous code range
     op_index, isa, discriminator, ...
```

```
   ┌─────────────┐  opcode   ┌──────────────┐  "append row"  ┌─────────────┐
   │   bytecode  │ ────────▶ │ update regs  │ ─────────────▶ │ line table  │
   │ (.debug_line)│          │ (addr,line..)│                │   row added │
   └─────────────┘           └──────────────┘                └─────────────┘
```

---

## 6.3.3 The three opcode classes

```
   1. SPECIAL opcodes (the workhorse, opcodes >= opcode_base):
      ONE byte that simultaneously advances `address` AND `line` AND appends a
      row. This is the compression magic — the common "next instruction, next
      line" case is a single byte.

   2. STANDARD opcodes (small, fixed set):
      DW_LNS_copy           append a row (no advance)
      DW_LNS_advance_pc     add to address
      DW_LNS_advance_line   add to line (signed)
      DW_LNS_set_file       set file register
      DW_LNS_set_column     set column
      DW_LNS_negate_stmt    flip is_stmt
      DW_LNS_const_add_pc   advance address by a constant
      DW_LNS_fixed_advance_pc

   3. EXTENDED opcodes (variable length, introduced by a 0 byte):
      DW_LNE_end_sequence   append row with end_sequence=1, then RESET the machine
      DW_LNE_set_address    set address to an absolute value (needs a reloc!)
      DW_LNE_define_file
```

### The special-opcode formula

A special opcode encodes both an address delta and a line delta in one byte:

```
   adjusted_opcode = opcode - opcode_base
   line_advance    = line_base + (adjusted_opcode % line_range)
   address_advance = (adjusted_opcode / line_range) * min_inst_length
   → apply both, then APPEND A ROW.
```

```
   Example (typical params: opcode_base=13, line_base=-5, line_range=14):
     a single byte like 0x4B → adjusted = 0x4B-13 = 62
       line_advance = -5 + (62 % 14) = -5 + 6 = +1     (advance 1 line)
       addr_advance = (62 / 14) * 1   = 4              (advance 4 bytes)
     → "4 bytes later, 1 line down, emit a row" in ONE byte.
```

---

## 6.3.4 A worked program trace

```
   .debug_line program (decoded operations):

   DW_LNE_set_address 0x1149      ; address = 0x1149   (the start of main; relocated)
   DW_LNS_set_file 1              ; file = demo.c
   special (line+=3, addr+=0)     ; line = 1+3 = 4, EMIT ROW: (0x1149, line 4)
   special (line+=1, addr+=8)     ; addr=0x1151, line=5, EMIT ROW: (0x1151, line 5)
   special (line+=1, addr+=15)    ; addr=0x1160, line=6, EMIT ROW: (0x1160, line 6)
   DW_LNS_advance_pc 16           ; addr=0x1170
   DW_LNE_end_sequence            ; EMIT ROW: (0x1170, end), reset machine

   Reconstructs exactly the table from §6.3.1, from a handful of bytes.
```

---

## 6.3.5 The `.debug_line` header

Before the program comes a header defining the parameters and the file/directory
tables:

```
   Line program header:
   ┌────────────────────────────────────────────────────────────┐
   │ unit_length, version                                       │
   │ minimum_instruction_length, maximum_ops_per_instruction    │
   │ default_is_stmt                                            │
   │ line_base, line_range, opcode_base   ← special-opcode math │
   │ standard_opcode_lengths[]                                  │
   │ directory table   (the include/search dirs)                │
   │ file_name table   (index → file, dir index, mtime, size)   │
   └────────────────────────────────────────────────────────────┘
   The `file` register indexes the file_name table; that's how a row's
   numeric file id becomes "demo.c".
```

> **DWARF5 change ▸** In DWARF ≤4 the file table is 1-based and the first real
> file is index 1; in DWARF5 it's 0-based and includes the primary source file at
> index 0, with names pulled from `.debug_line_str`. A frequent off-by-one when
> writing parsers.

---

## 6.3.6 `is_stmt`, breakpoints, and "is this a good line?"

The `is_stmt` flag marks rows that are **recommended breakpoint locations** —
the start of a statement, where setting a breakpoint makes sense and the program
state is consistent. GDB's `break file:line` snaps to an `is_stmt` row.

```
   one source line can map to MANY addresses; is_stmt picks the canonical one:
     line 5:  0x1151 (is_stmt) ← breakpoint goes here
              0x1156           ← mid-statement, not a breakpoint target
              0x115b           ← ditto
```

The `discriminator` register distinguishes multiple blocks on the same line
(e.g. both sides of a ternary, or a macro expanded multiple times) — important
for accurate profiling and coverage.

---

## 6.3.7 Why `DW_LNE_set_address` needs a relocation

`DW_LNE_set_address 0x1149` embeds an absolute code address. In a relocatable
`.o`, that address isn't final yet — so the `.debug_line` section carries a
**relocation** (`.rela.debug_line`) pointing at that operand, which the linker
patches just like code relocations (Part 4.6). This is why even debug sections
have relocations.

```bash
readelf -rW demo.o | grep debug_line    # relocations into the line program
```

---

## 6.3.8 Reading line tables

**Try it ▸**

```bash
gcc -g -O0 demo.c -o demo

# The decoded, human-readable table:
readelf --debug-dump=decodedline demo

# The raw program opcodes (see the special/standard/extended ops):
readelf --debug-dump=rawline demo | head -60
llvm-dwarfdump --debug-line demo | head -60

# Use it the way tools do — address to line and back:
addr2line -e demo -f -i 0x1149          # -i shows inlined frames too
gdb -batch -ex 'info line *0x1149' demo
```

The decoded output is literally the table from §6.3.1; the raw output is the
bytecode that generates it. Seeing both side by side makes the state machine
click.

---

## 6.3.9 How this powers everyday tools

```
   crash address 0x563...1162
        │ subtract load base (ASLR) → file-relative 0x1162
        ▼
   .debug_aranges:  0x1162 ∈ CU "demo.c"          (which CU?)
        ▼
   .debug_line:     run the program, find the row whose address range
                    contains 0x1162  →  demo.c, line 6
        ▼
   "Segfault at demo.c:6"     (what GDB / addr2line / a backtrace prints)
```

Profilers (`perf`), coverage tools (`gcov`), and sanitizers all lean on this
exact mechanism to attribute machine addresses to source lines.

---

## Summary

- `.debug_line` stores a **bytecode program** that regenerates the
  address→(file,line,column,flags) table, exploiting the mapping's incremental
  nature for huge compression.
- A **state machine** (address, file, line, is_stmt, …) is driven by **special**
  (one byte = advance address+line+emit row), **standard**, and **extended**
  (`set_address`, `end_sequence`) opcodes.
- The header defines the special-opcode math (`line_base`, `line_range`,
  `opcode_base`) and the file/dir tables; DWARF5 changed file indexing to 0-based.
- `is_stmt` marks breakpoint-worthy rows; `discriminator` separates same-line
  blocks for accurate profiling.
- `DW_LNE_set_address` carries a relocation so the linker can fix up code
  addresses in the debug line table.
- This machinery is what `addr2line`, GDB backtraces, `perf`, and coverage tools
  use to map addresses to source.

Next: [6.4 — Call frame information & stack unwinding](04-cfi-and-unwinding.md)
