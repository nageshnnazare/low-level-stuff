# 6.1 — DWARF Overview & Sections

DWARF is the debugging-information format that lets a debugger map between the
*machine* (addresses, registers, raw bytes) and the *source* (functions,
variables, types, line numbers). It is how GDB knows that address `0x1149` is
`main` at `demo.c:5`, that `rbx` currently holds the variable `count`, and how to
unwind the stack. This part dissects it.

---

## 6.1.1 What DWARF is (and the name)

DWARF ("Debugging With Attributed Record Formats") is a companion to ELF (a
playful medieval pun — ELF/DWARF). It's an open standard (current: DWARF 5,
2017). It lives in `.debug_*` sections of the ELF file but is **not loaded at
runtime** (not `SHF_ALLOC`); only debuggers, profilers, and unwinders read it.

```
   COMPILE-TIME knowledge                       RUNTIME reality
   ─────────────────────                        ───────────────
   "variable count is an int,                   register rbx = 0x2a
    a local of main, scoped              ◀DWARF▶ instruction pointer = 0x1162
    to lines 5-9, currently in rbx"              memory at 0x7fff... = ...

   DWARF is the dictionary that translates between these two worlds.
```

Without DWARF you can still run a program (debug info is optional), but a
debugger shows only addresses and raw assembly — no source lines, no variable
names, no types.

---

## 6.1.2 The design goals that shape DWARF

DWARF is dense and complex because it must:

```
   1. Describe ANY language (C, C++, Rust, Fortran, Ada, ...) — it's language-agnostic.
   2. Survive AGGRESSIVE OPTIMIZATION — describe variables that live in different
      registers at different PCs, that are split, duplicated, or optimized away.
   3. Be COMPACT — debug info often dwarfs (!) the code; heavy compression via
      LEB128, abbreviations, string pooling, and a tree of shared types.
   4. Allow FAST lookup — indexed by address (.debug_aranges) and name
      (.debug_names) so a debugger doesn't scan everything.
```

The "describe optimized code" requirement is why DWARF has *location
expressions* (a tiny stack machine) instead of "variable X is always at offset
Y" — a variable's location can change per instruction.

---

## 6.1.3 The `.debug_*` sections

```
 ┌──────────────────┬─────────────────────────────────────────────────────────┐
 │ Section          │ Contents                                                │
 ├──────────────────┼─────────────────────────────────────────────────────────┤
 │ .debug_info      │ the core tree of DIEs (functions, vars, types) — §6.2   │
 │ .debug_abbrev    │ abbreviation tables: schemas for the DIEs in .debug_info│
 │ .debug_str       │ string pool for names (offset-referenced)               │
 │ .debug_line      │ the line-number program (address ↔ file:line) — §6.3    │
 │ .debug_line_str  │ string pool for the line program (DWARF5)               │
 │ .debug_loclists  │ location lists: where a variable lives, per PC range    │
 │ .debug_ranges/   │ non-contiguous address ranges for scopes                │
 │   .debug_rnglists│   (rnglists is the DWARF5 form)                         │
 │ .debug_addr      │ address pool (DWARF5, for split debug)                  │
 │ .debug_str_offsets│ indirection table for strings (DWARF5)                 │
 │ .debug_aranges   │ address → compilation unit fast index                   │
 │ .debug_names /   │ name → DIE fast index (DWARF5 / GNU's .gdb_index)       │
 │   .gdb_index     │                                                         │
 │ .debug_frame /   │ Call Frame Information for unwinding — §6.4             │
 │   .eh_frame      │ (.eh_frame is the ALLOC'd, runtime-usable form)         │
 │ .debug_macro     │ macro definitions (#define) for expression eval         │
 └──────────────────┴─────────────────────────────────────────────────────────┘
```

```
   How they reference each other:
   .debug_info ──abbrev codes──▶ .debug_abbrev   (how to parse each DIE)
   .debug_info ──string offsets─▶ .debug_str / .debug_line_str
   .debug_info ──loc offsets────▶ .debug_loclists (variable locations)
   .debug_info ──stmt_list──────▶ .debug_line     (this CU's line table)
   .debug_info ──addr_base──────▶ .debug_addr      (address pool)
```

> **`.eh_frame` is special ▸** Unlike the other `.debug_*` sections, `.eh_frame`
> **is** allocated (loaded at runtime), because C++ exception handling needs to
> unwind the stack at runtime, not just in a debugger. So even a *stripped*,
> non-debug C++ binary still carries `.eh_frame`. More in §6.4.

**Try it ▸**

```bash
gcc -g -O0 demo.c -o demo
readelf -S demo | grep debug          # the .debug_* sections
size --format=sysv demo | grep -i debug
# How much is debug info? (often > the code!)
objcopy --only-keep-debug demo demo.dbg && ls -l demo demo.dbg
```

---

## 6.1.4 DWARF versions and what changed

```
   DWARF 2 (1993):  the foundation — DIEs, line program, CFI.
   DWARF 3 (2005):  64-bit, namespaces, better C++ support.
   DWARF 4 (2010):  .debug_types, improved location lists, type units.
   DWARF 5 (2017):  .debug_loclists/.debug_rnglists/.debug_addr/.debug_names,
                    split DWARF, .debug_line_str, more compact forms.
```

Pick the version with `-gdwarf-5` (GCC/Clang default is 4 or 5 depending on
version). DWARF 5 is more compact and is the modern target. `readelf
--debug-dump` and `llvm-dwarfdump` handle all versions.

---

## 6.1.5 Split DWARF and debug fission (keeping builds fast)

Debug info is *huge* — often several times the size of the code — and it bloats
object files and slows linking. **Split DWARF** (`-gsplit-dwarf`) moves the bulk
into separate `.dwo` (DWARF object) files that the linker never touches:

```
   normal:   foo.o   = code + ALL debug info        → linker reads it all
   split:    foo.o   = code + skeleton + .debug_addr  (tiny)
             foo.dwo = the heavy DIEs, types, strings  (linker ignores)

   ┌────────────┐         ┌──────────────────────────┐
   │ foo.o      │────────▶│linker (fast: small input)│──▶ a.out (+ skeleton)
   │ +skeleton  │         └──────────────────────────┘
   └────────────┘
   ┌────────────┐
   │ foo.dwo    │◀── debugger fetches the full info ON DEMAND, by reference
   └────────────┘
```

This dramatically speeds linking (the linker doesn't relocate gigabytes of debug
info). `dwp` packages `.dwo`s into a single `.dwp`. Used at scale by Chromium and
similar giant C++ projects.

---

## 6.1.6 Separate debug files and the build-id link

Distributions ship stripped binaries plus *separate* debug files, linked by a
**build-id** (a hash in `.note.gnu.build-id`):

```
   /usr/bin/app                       (stripped; has .note.gnu.build-id = abc123...)
   /usr/lib/debug/.build-id/ab/c123...debug   (the matching DWARF)

   gdb reads the build-id from the running binary, then locates the debug file
   by that id. `debuginfod` servers serve these on demand over HTTP.
```

```bash
# The standard split:
objcopy --only-keep-debug app app.debug      # extract debug
strip --strip-debug app                       # shrink the shipped binary
objcopy --add-gnu-debuglink=app.debug app     # record the link
readelf -n app | grep -i build                # the build-id
```

This is why you can `apt install <pkg>-dbgsym` (or use `debuginfod`) to debug a
release binary with full symbols, without the production binary carrying them.

---

## 6.1.7 Your DWARF tools

```
   readelf --debug-dump=info      DIEs (the type/var/function tree)
   readelf --debug-dump=abbrev    abbreviation tables
   readelf --debug-dump=decodedline   address ↔ source line table (readable)
   readelf --debug-dump=frames    CFI / unwind tables
   llvm-dwarfdump --all FILE      LLVM's excellent DWARF dumper (very readable)
   dwarfdump FILE                 the libdwarf reference dumper
   addr2line -e FILE -f 0x1149    address → function + file:line (uses DWARF)
   gdb                            consumes all of it for interactive debugging
```

**Try it ▸** The "hello DWARF" command everyone should run once:

```bash
llvm-dwarfdump --debug-info demo | head -40    # the DIE tree for demo.c
addr2line -e demo -f 0x1149                      # which source line is this?
```

---

## Summary

- DWARF is the language-agnostic debug format that translates between machine
  state and source-level concepts; it lives in non-loaded `.debug_*` ELF
  sections (except `.eh_frame`, which **is** loaded for C++ unwinding).
- It's designed to describe any language, survive heavy optimization (via
  location *expressions*), stay compact (LEB128, abbreviations, string pools),
  and support fast address/name lookup.
- Core sections: `.debug_info` (DIEs) + `.debug_abbrev` (schemas), `.debug_line`
  (line table), `.debug_loclists` (variable locations), `.debug_str`,
  `.eh_frame`/`.debug_frame` (unwinding).
- DWARF 5 is the modern, compact version; **split DWARF** (`.dwo`/`.dwp`) and
  **separate debug files** (build-id, debuginfod) keep large builds fast and
  release binaries small.

Next: [6.2 — DIEs: the debug information tree](02-dies-and-debug-info.md)
