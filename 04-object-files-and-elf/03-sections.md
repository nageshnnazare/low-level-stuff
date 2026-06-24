# 4.3 — Sections & the Section Header Table

Sections are the **linking view** of an ELF file: the fine-grained, named pieces
the linker, assembler, and debuggers work with. This chapter dissects the
section header structure, the standard sections, and the special index values.

---

## 4.3.1 The `Elf64_Shdr` structure

The section header table is an array of these, located at `e_shoff`, with
`e_shnum` entries each `e_shentsize` (64) bytes:

```c
typedef struct {
    Elf64_Word  sh_name;       /*  0  offset into .shstrtab for the name   */
    Elf64_Word  sh_type;       /*  4  section type (SHT_*)                  */
    Elf64_Xword sh_flags;      /*  8  attribute flags (SHF_*)              */
    Elf64_Addr  sh_addr;       /* 16  virtual addr at runtime (0 if none)  */
    Elf64_Off   sh_offset;     /* 24  file offset of the section's bytes    */
    Elf64_Xword sh_size;       /* 32  size in bytes                         */
    Elf64_Word  sh_link;       /* 40  link to another section (type-dependent)*/
    Elf64_Word  sh_info;       /* 44  extra info (type-dependent)          */
    Elf64_Xword sh_addralign;  /* 48  required alignment                   */
    Elf64_Xword sh_entsize;    /* 56  size of each entry, if a table        */
} Elf64_Shdr;                  /* 64 bytes                                  */
```

```
   sh_offset ──────────▶ where the bytes are IN THE FILE
   sh_addr   ──────────▶ where the bytes go IN MEMORY (if SHF_ALLOC)
   sh_size   ──────────▶ how many bytes
   sh_link/sh_info ────▶ cross-references (e.g. .symtab → its .strtab)
   sh_entsize ─────────▶ for tables: stride (e.g. 24 for Elf64_Sym)
```

> **`sh_offset` vs `sh_addr` ▸** This pair is the file↔memory mapping at section
> granularity. For `SHT_NOBITS` (`.bss`), `sh_offset` points "where it would be"
> but no bytes are actually stored; `sh_addr` and `sh_size` still describe its
> runtime footprint.

---

## 4.3.2 Section types (`sh_type`)

```
 ┌───────────────────┬──────────────────────────────────────────────────┐
 │ SHT_NULL     (0)  │ inactive; section header 0 is always NULL        │
 │ SHT_PROGBITS (1)  │ program-defined bytes (.text, .data, .rodata, ..)│
 │ SHT_SYMTAB   (2)  │ symbol table (full, for linking)                 │
 │ SHT_STRTAB   (3)  │ string table                                     │
 │ SHT_RELA     (4)  │ relocations WITH addends (.rela.*)               │
 │ SHT_HASH     (5)  │ symbol hash table (dynamic linking)              │
 │ SHT_DYNAMIC  (6)  │ the .dynamic array (dynamic linking info)        │
 │ SHT_NOTE     (7)  │ notes (build-id, ABI tags, GNU properties)       │
 │ SHT_NOBITS   (8)  │ occupies no file space (.bss, .tbss)             │
 │ SHT_REL      (9)  │ relocations WITHOUT addends (.rel.*; i386)       │
 │ SHT_DYNSYM  (11)  │ dynamic symbol table (subset, for the loader)    │
 │ SHT_INIT_ARRAY(14)│ array of constructor pointers                    │
 │ SHT_FINI_ARRAY(15)│ array of destructor pointers                     │
 │ SHT_GROUP   (17)  │ a COMDAT/section group (C++ template dedup)      │
 │ SHT_GNU_HASH(0x6ffffff6)│ GNU's faster dynamic symbol hash           │
 └───────────────────┴──────────────────────────────────────────────────┘
```

> **Two symbol tables ▸** `.symtab` (`SHT_SYMTAB`) is the *complete* symbol
> table used at link time; it can be stripped. `.dynsym` (`SHT_DYNSYM`) is the
> *minimal* table needed for dynamic linking at runtime; it survives stripping
> because the loader needs it. Same struct, different audience. (See §4.5.)

---

## 4.3.3 Section flags (`sh_flags`)

```
   SHF_WRITE      (0x1)   writable at runtime
   SHF_ALLOC      (0x2)   occupies memory during execution
   SHF_EXECINSTR  (0x4)   executable machine code
   SHF_MERGE      (0x10)  may be merged to eliminate duplicates
   SHF_STRINGS    (0x20)  contains NUL-terminated strings
   SHF_INFO_LINK  (0x40)  sh_info holds a section index
   SHF_GROUP      (0x200) member of a section group (COMDAT)
   SHF_TLS        (0x400) thread-local storage
```

```
   The flags letters readelf prints map to these:
   W=WRITE  A=ALLOC  X=EXECINSTR  M=MERGE  S=STRINGS  G=GROUP  T=TLS  I=INFO_LINK
```

These flags are what the linker uses to decide which output segment a section
belongs in. `SHF_ALLOC` is the master switch for "loaded at runtime."

> **SHF_MERGE + SHF_STRINGS ▸** `.rodata.str1.1` carries both flags, telling the
> linker it may **deduplicate** identical string literals across all input
> objects. This is why two source files that both contain `"error: "` end up
> sharing one copy in the final binary. `sh_entsize` = the element size for
> mergeable fixed-size constants.

---

## 4.3.4 `sh_link` and `sh_info`: the cross-reference fields

These two fields mean different things per section type — a frequent source of
confusion. The table:

```
 ┌──────────────┬───────────────────────────┬────────────────────────────────┐
 │ Section type │ sh_link                   │ sh_info                        │
 ├──────────────┼───────────────────────────┼────────────────────────────────┤
 │ SHT_SYMTAB / │ index of associated       │ index of first non-local       │
 │ SHT_DYNSYM   │   string table section    │ symbol (= # of local syms)     │
 │ SHT_RELA /   │ index of the symbol table │ index of the section the relocs│
 │ SHT_REL      │   the relocs refer to     │   apply TO                     │
 │ SHT_DYNAMIC  │ index of its string table │ 0                              │
 │ SHT_HASH     │ index of its symbol table │ 0                              │
 │ SHT_GROUP    │ index of the symtab       │ index of the group's signature │
 │              │                           │   symbol                       │
 └──────────────┴───────────────────────────┴────────────────────────────────┘
```

```
   Example chain for a relocation section:

   .rela.text  ──sh_info──▶ .text     ("apply my relocs to .text")
        │
        └────sh_link──▶ .symtab       ("my r_info sym indices are into .symtab")
                            │
                            └──sh_link──▶ .strtab  ("symbol names are here")
```

> **`sh_info` = first global symbol ▸** ELF requires that a symbol table list
> all LOCAL symbols *before* any GLOBAL/WEAK ones. `sh_info` records the
> boundary, so the linker can skip locals quickly. If you ever hand-craft a
> symtab and get this wrong, linkers misbehave subtly.

---

## 4.3.5 Special section indices

Some section indices are reserved sentinels (they appear as a symbol's
`st_shndx`, see §4.5):

```
   SHN_UNDEF  (0)       undefined / not present — references to external symbols
   SHN_ABS    (0xfff1)  symbol value is absolute, not affected by relocation
   SHN_COMMON (0xfff2)  unallocated common (tentative) symbol
   SHN_XINDEX (0xffff)  real index is elsewhere (>= 0xff00 sections)
   SHN_LORESERVE..HIRESERVE (0xff00..0xffff)  reserved range
```

Section header **index 0** is always a `SHT_NULL` entry of all zeros — a
universal "none" sentinel so that index 0 can mean "no section."

---

## 4.3.6 Reading the section header table

**Try it ▸**

```bash
readelf -SW demo
```

Annotated output (typical PIE):

```
 [Nr] Name              Type      Address    Off    Size   ES Flg Lk Inf Al
 [ 0]                   NULL      0          0      0      00      0   0  0   ← null
 [ 1] .interp           PROGBITS  0x318      0x318  0x1c   00  A   0   0  1   ← loader path
 [ 2] .note.gnu.property NOTE     ...                          A
 [ 3] .note.gnu.build-id NOTE     ...                          A           ← build-id
 [ 4] .gnu.hash         GNU_HASH  ...                          A   6       ← dynsym hash
 [ 5] .dynsym           DYNSYM    ...                          A   6   1    ← runtime symbols
 [ 6] .dynstr           STRTAB    ...                          A
 [ 7] .gnu.version      VERSYM    ...
 [ 8] .gnu.version_r    VERNEED   ...
 [ 9] .rela.dyn         RELA      ...                          A   5       ← load-time relocs
 [10] .rela.plt         RELA      ...                          AI  5  24   ← PLT relocs
 [11] .init             PROGBITS  ...                          AX          ← _init
 [12] .plt              PROGBITS  ...                          AX          ← PLT stubs
 [13] .text             PROGBITS  0x1050     ...               AX          ← your code
 [14] .fini             PROGBITS  ...                          AX
 [15] .rodata           PROGBITS  ...                          A           ← constants
 [16] .eh_frame_hdr     PROGBITS  ...                          A           ← unwind index
 [17] .eh_frame         PROGBITS  ...                          A           ← unwind/CFI
 [18] .init_array       INIT_ARRAY...                          WA          ← global ctors
 [19] .fini_array       FINI_ARRAY...                          WA
 [20] .dynamic          DYNAMIC   ...                          WA   6
 [21] .got              PROGBITS  ...                          WA          ← GOT
 [22] .got.plt          PROGBITS  ...                          WA          ← lazy-bind GOT
 [23] .data             PROGBITS  ...                          WA          ← init data
 [24] .bss              NOBITS    ...                          WA          ← zero data
 [25] .comment          PROGBITS  ...                              ← toolchain tag
 [26] .debug_*          PROGBITS  ...                              ← DWARF (if -g)
 [..] .symtab           SYMTAB    ...                          ← full symtab (Lk=.strtab)
 [..] .strtab           STRTAB
 [..] .shstrtab         STRTAB                                 ← section name strings
```

Walk the `Flg` column: `AX` sections (alloc+exec) are code; `WA` are writable
data; no flag (`.symtab`, `.debug_*`, `.shstrtab`) means not loaded at runtime.

---

## 4.3.7 How section names are resolved (the .shstrtab dance)

```
   1. ELF header → e_shstrndx (say 30)
   2. Section header [30] is .shstrtab (SHT_STRTAB)
   3. .shstrtab bytes:  \0 . s y m t a b \0 . s t r t a b \0 . t e x t \0 ...
   4. For section [13], sh_name = (offset of ".text" in .shstrtab)
   5. Read NUL-terminated string at that offset → ".text"
```

```
   .shstrtab:  \0  .text\0  .data\0  .bss\0  .rodata\0 ...
   offset:     0   1        7        13      18
   section [13].sh_name = 1   → ".text"
   section [23].sh_name = 7   → ".data"
```

This same pattern (a section of NUL-separated strings, referenced by byte
offset) recurs for `.strtab` (symbol names) and `.dynstr`.

---

## 4.3.8 Section alignment and layout in the file

The linker places sections in the file/memory respecting `sh_addralign`. It pads
between sections. Within a `PT_LOAD` segment, sections are ordered so that
permission-compatible sections are contiguous (so the segment has one
permission). This is why `.text` and `.rodata` sit together (both read-only) and
`.data`/`.bss` together (both writable). We connect this to segments in §4.4.

```
   file:  ...[.text  pad][.rodata  pad][.data  pad][.bss size-only]...
                │ align 16    │ align 8     │ align 8
```

---

## Summary

- The section header table (`e_shoff`, `e_shnum` × 64-byte `Elf64_Shdr`) is the
  linking view's index.
- Each header carries name, type, flags, file offset, runtime address, size, two
  cross-reference fields (`sh_link`/`sh_info`), alignment, and entry stride.
- `sh_type` distinguishes PROGBITS/NOBITS/SYMTAB/STRTAB/RELA/DYNAMIC/etc.;
  `sh_flags` (`A`/`W`/`X`/`M`/`S`/`T`/`G`) drive segment assignment and merging.
- `sh_link`/`sh_info` wire sections together (relocs→symtab→strtab); their
  meaning is type-dependent — memorize the table.
- Special indices (`SHN_UNDEF`, `SHN_ABS`, `SHN_COMMON`) and the null section 0
  are sentinels.
- Section names live in `.shstrtab`, resolved via `e_shstrndx` + `sh_name`.

Next: [4.4 — Segments & the program header table](04-segments.md)
