# 4.1 — ELF Overview & the Dual Nature of the Format

ELF (**Executable and Linkable Format**) is the universal container on Linux,
BSD, Solaris, and most embedded systems. Object files, executables, shared
libraries, and core dumps are all ELF. Mastering ELF means you can read any of
them. This part dissects it down to the byte.

---

## 4.1.1 What ELF is and where it lives

ELF was designed at AT&T for System V and standardized in the **System V ABI**
(generic ELF spec) plus a per-architecture supplement (the "psABI", e.g. the
*x86-64 psABI*). Four file types share the format:

```
 ┌─────────────┬──────────┬────────────────────────────────────────────┐
 │ e_type      │ Name     │ What it is                                 │
 ├─────────────┼──────────┼────────────────────────────────────────────┤
 │ ET_REL  (1) │ .o, .a   │ relocatable object — input to the linker   │
 │ ET_EXEC (2) │ a.out    │ executable, fixed load address (non-PIE)   │
 │ ET_DYN  (3) │ .so, PIE │ shared object OR position-independent exe  │
 │ ET_CORE (4) │ core     │ core dump (process snapshot for debugging) │
 └─────────────┴──────────┴────────────────────────────────────────────┘
```

> **PIE is ET_DYN ▸** A modern "executable" is usually `ET_DYN` (a
> position-independent executable), not `ET_EXEC`. That's why `file ./a.out`
> says "shared object" on Ubuntu. The kernel/loader can place it at a randomized
> base (ASLR). True `ET_EXEC` (fixed address) is now the exception. See §4.2.

Non-Linux equivalents (same ideas, different container):

```
   ELF        ←→   Mach-O (macOS/iOS)     ←→   PE/COFF (Windows)
   sections   ←→   sections in segments   ←→   sections
   ld.so      ←→   dyld                    ←→   the PE loader
   DWARF      ←→   DWARF (in .dSYM)        ←→   PDB / CodeView
```

---

## 4.1.2 The dual nature: linking view vs execution view

This is *the* mental model for ELF. The same file is described two ways, by two
different tables:

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                            ELF FILE                                    │
 │  ┌──────────────────────────────────────────────────────────────────┐  │
 │  │  ELF Header        (always at offset 0; the map to everything)   │  │
 │  └──────────────────────────────────────────────────────────────────┘  │
 │                                                                        │
 │  ┌──────── LINKING VIEW ─────────┐    ┌─────── EXECUTION VIEW ───────┐ │
 │  │  Section Header Table         │    │  Program Header Table        │ │
 │  │  (array of Elf64_Shdr)        │    │  (array of Elf64_Phdr)       │ │
 │  │                               │    │                              │ │
 │  │  describes SECTIONS:          │    │  describes SEGMENTS:         │ │
 │  │   .text .rodata .data .bss    │    │   PT_LOAD (r-x)              │ │
 │  │   .symtab .strtab .rela.*     │    │   PT_LOAD (rw-)              │ │
 │  │   .debug_* ...                │    │   PT_INTERP PT_DYNAMIC ...   │ │
 │  │                               │    │                              │ │
 │  │  Used by: ld, readelf -S,     │    │  Used by: kernel, ld.so,     │ │
 │  │           objdump, debuggers  │    │           readelf -l         │ │
 │  └───────────────────────────────┘    └──────────────────────────────┘ │
 │                                                                        │
 │  ┌────────────────────────────────────────────────────────────────┐    │
 │  │  The actual bytes: code, data, symbol tables, strings, ...     │    │
 │  │  (sections and segments are just two ways to *index* these)    │    │
 │  └────────────────────────────────────────────────────────────────┘    │
 └────────────────────────────────────────────────────────────────────────┘
```

The same byte ranges are referenced by both tables. A single `PT_LOAD` segment
(execution view) typically *contains* several sections (linking view):

```
   PT_LOAD (r-x) segment  ⊇  { .init, .plt, .text, .fini, .rodata, .eh_frame }
   PT_LOAD (rw-) segment  ⊇  { .init_array, .data, .bss, .got, .dynamic }
```

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │ FILE LAYOUT (offsets) and how sections map into LOAD segments       │
 │                                                                     │
 │  off 0     ┌──────────────┐                                         │
 │            │ ELF header   │                                         │
 │            │ Program hdrs │ ◀── described by Phdr table             │
 │            ├──────────────┤  ┐                                      │
 │            │ .text        │  │                                      │
 │            │ .rodata      │  ├── PT_LOAD #1 (r-x)                   │
 │            │ .eh_frame    │  │                                      │
 │            ├──────────────┤  ┘                                      │
 │            │ .data        │  ┐                                      │
 │            │ .got/.dynamic│  ├── PT_LOAD #2 (rw-)                   │
 │            │ .bss (NOBITS)│  │   (.bss has size but no file bytes)  │
 │            ├──────────────┤  ┘                                      │
 │            │ .symtab      │  ┐                                      │
 │            │ .strtab      │  ├── NOT in any PT_LOAD                 │
 │            │ .debug_*     │  │   (not loaded at runtime)            │
 │            │ Section hdrs │  ┘                                      │
 │            └──────────────┘                                         │
 └─────────────────────────────────────────────────────────────────────┘
```

Key consequences:

- A **relocatable object** (`.o`) has only the linking view that matters
  (sections); its program header table is empty/absent.
- A **loaded executable/shared object** needs only the execution view at
  runtime; you can `strip` away section headers and `.symtab` and it still runs.
- Some bytes (debug, symtab) are in the file but in **no** segment → never
  occupy runtime memory.

---

## 4.1.3 The five structures you must know

Everything in ELF is built from five C structures (64-bit variants shown). The
rest of Part 4 takes each in turn.

```
   1. Elf64_Ehdr   — the ELF header (file offset 0)         → §4.2
   2. Elf64_Shdr   — a section header (array = SHT)         → §4.3
   3. Elf64_Phdr   — a program header (array = PHT)         → §4.4
   4. Elf64_Sym    — a symbol table entry                   → §4.5
   5. Elf64_Rela   — a relocation entry                     → §4.6
```

```
                       ┌─ e_shoff ──▶ Section Header Table ─▶ each Shdr ─▶ a section's bytes
   Elf64_Ehdr (off 0) ─┤
                       └─ e_phoff ──▶ Program Header Table ─▶ each Phdr ─▶ a segment's bytes

   .symtab section ──▶ array of Elf64_Sym ──▶ st_name indexes .strtab
   .rela.text section ─▶ array of Elf64_Rela ─▶ r_info indexes .symtab
```

The ELF header is the root of this graph: it tells you where the section header
table and program header table are, and everything else is reachable from there.

---

## 4.1.4 ELF's portable type system

ELF defines fixed-width types so the format is identical regardless of the
host's C compiler (recall the LP64/LLP64 hazard from §1.2.1):

```
 ┌──────────────┬──────┬────────────────────────────────────┐
 │ Type         │ Size │ Meaning                            │
 ├──────────────┼──────┼────────────────────────────────────┤
 │ Elf64_Addr   │ 8    │ a virtual address                  │
 │ Elf64_Off    │ 8    │ a file offset                      │
 │ Elf64_Half   │ 2    │ small int (e.g. e_type, e_machine) │
 │ Elf64_Word   │ 4    │ 32-bit unsigned                    │
 │ Elf64_Sword  │ 4    │ 32-bit signed                      │
 │ Elf64_Xword  │ 8    │ 64-bit unsigned                    │
 │ Elf64_Sxword │ 8    │ 64-bit signed                      │
 │ unsigned char│ 1    │ byte                               │
 └──────────────┴──────┴────────────────────────────────────┘
```

The 32-bit ELF variants (`Elf32_*`) use 4-byte `Addr`/`Off` and reorder some
struct fields. The `EI_CLASS` byte in the header tells you which to use. We focus
on 64-bit; the concepts are identical.

---

## 4.1.5 Your ELF microscope: readelf, objdump, and raw bytes

Three lenses, used throughout Part 4:

```
   readelf  — purpose-built ELF dumper, follows the spec field names exactly.
              Best for STRUCTURE.   (readelf -h / -S / -l / -s / -r / -d ...)
   objdump  — disassembler-centric, BFD-based.
              Best for CODE + relocations.  (objdump -d / -dr / -t / -R ...)
   xxd/od   — raw bytes. The ground truth when tools disagree or for learning.
```

**Try it ▸** Build one binary and look at it through all three lenses (you'll
reuse this binary across the whole part):

```bash
cat > demo.c <<'EOF'
#include <stdio.h>
int    g_init = 0x11223344;     /* .data  */
int    g_zero;                  /* .bss   */
const char *msg = "hello elf";  /* ptr in .data, string in .rodata */
int main(void){ printf("%s %x %d\n", msg, g_init, g_zero); return 0; }
EOF
gcc -g -O0 demo.c -o demo

file demo
readelf -h demo            # the header (next chapter)
readelf -S demo            # sections
readelf -l demo            # segments
xxd demo | head -4         # raw magic
```

---

## 4.1.6 A note on ELF's longevity and design

ELF has survived since 1999 largely unchanged because its design is **extensible
by convention**: new section *names*, new note types, new dynamic tags, and new
relocation types can be added without breaking old tools (which ignore what they
don't recognize). DWARF, build-ids, GNU properties, RELRO, IFUNCs, and TLS were
all bolted on this way. This is why you'll see many `GNU_*` and `SHT_GNU_*`
extensions beyond the base spec.

---

## Summary

- ELF is the universal container for objects, executables, shared libraries, and
  core dumps; type is in `e_type` (PIE executables are `ET_DYN`).
- The **dual nature** is the master concept: the **section header table**
  (linking view) and **program header table** (execution view) describe the same
  bytes for different consumers (linker vs loader).
- Some bytes belong to no segment and are never loaded (symtab, debug).
- Five structures — Ehdr, Shdr, Phdr, Sym, Rela — compose all of ELF, rooted at
  the ELF header.
- ELF uses fixed-width portable types and is extensible by convention.

Next: [4.2 — The ELF header, byte by byte](02-elf-header.md)
