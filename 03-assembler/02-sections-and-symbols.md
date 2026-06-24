# 3.2 — Sections, the Location Counter & Symbols

Sections and symbols are the two abstractions the assembler hands to the linker.
Getting them precisely right is the foundation for everything in Parts 4 and 5.

---

## 3.2.1 What a section *is*

A **section** is a named, contiguous blob of bytes (or, for `.bss`, a named *size*
with no bytes) plus attributes describing how it should be treated. The
assembler groups everything it emits into sections; the linker later merges
same-named sections and maps them to memory segments.

```
   A section =  name  +  flags  +  type  +  alignment  +  the bytes
                 │       │         │         │            │
              ".text"  AX     PROGBITS    16         <machine code>
```

### The standard sections

```
 ┌────────────┬──────────┬────────────────────────────────────────────────┐
 │ Section    │ Flags    │ Contents                                       │
 ├────────────┼──────────┼────────────────────────────────────────────────┤
 │ .text      │ A X      │ executable machine code                        │
 │ .rodata    │ A        │ read-only constants, string literals           │
 │ .data      │ A W      │ initialized read-write globals/statics         │
 │ .bss       │ A W      │ zero-init data(NOBITS:size only, no file bytes)│
 │ .tdata     │ A W T    │ initialized thread-local data                  │
 │ .tbss      │ A W T    │ zero-init thread-local data (NOBITS)           │
 │ .symtab    │ —        │ symbol table                                   │
 │ .strtab    │ —        │ strings for .symtab                            │
 │ .shstrtab  │ —        │ strings for section *names*                    │
 │ .rela.text │ —        │ relocations to apply to .text                  │
 │ .comment   │ —        │ toolchain version tags                         │
 │ .note.*    │ A        │ notes (build-id, ABI tag, ...)                 │
 │ .init_array│ A W      │ pointers to global ctors (run at startup)      │
 │ .fini_array│ A W      │ pointers to global dtors                       │
 │ .eh_frame  │ A        │ unwinding/CFI tables (Part 6.4)                │
 │ .debug_*   │ —        │ DWARF debug info (Part 6)                      │
 └────────────┴──────────┴────────────────────────────────────────────────┘
   Flags:  A=alloc (occupies memory at runtime)  W=writable  X=executable
           T=thread-local  M=mergeable  S=contains strings  G=group(COMDAT)
```

> **NOBITS vs PROGBITS ▸** `.bss` has type `SHT_NOBITS`: it occupies *runtime*
> memory but **zero** file bytes — the file just records its size, and the loader
> zero-fills. This is why a program with a huge zero-initialized global array has
> a small file. `.data` is `SHT_PROGBITS`: its bytes live in the file.

---

## 3.2.2 Why sections exist: separation by property

Sections group bytes that share *runtime properties* so the linker can give each
group the right memory permissions:

```
   "alloc + execute, read-only"   → .text     → mapped r-x
   "alloc, read-only"             → .rodata   → mapped r--   (often merged w/ .text seg)
   "alloc, read-write, has bytes" → .data     → mapped rw-
   "alloc, read-write, zero"      → .bss      → mapped rw-, zero-filled
   "not alloc"                    → .symtab,  → present in file, NOT in memory
                                     .debug_*    at runtime
```

The split between "alloc" and "not alloc" is fundamental: **debug info and
symbol tables are not loaded into memory when the program runs.** That's why you
can `strip` them out to shrink the runtime footprint without changing behavior.

```
   FILE                          MEMORY (at runtime)
   ─────                         ───────────────────
   .text     ──map──────────────▶ r-x  ┐
   .rodata   ──map──────────────▶ r--  ┘ loaded
   .data     ──map──────────────▶ rw-
   .bss      ──(size only)──────▶ rw- (zeroed)
   .symtab   ──┐
   .strtab   ──┤ NOT mapped — only the linker/debuggers read these from the file
   .debug_*  ──┘
```

---

## 3.2.3 Custom sections and why compilers make so many

Modern compilers emit *many* fine-grained sections so the linker can discard or
reorder them:

```asm
   .section .text.foo,"axG",@progbits,foo,comdat   # one section per function
   .section .text.unlikely,"ax",@progbits          # cold code
   .section .data.rel.ro,"aw",@progbits            # const-but-needs-relocation
```

- `-ffunction-sections` / `-fdata-sections`: put each function/variable in its
  own section (`.text.foo`, `.data.bar`). This lets the linker do
  `--gc-sections` (garbage-collect unused functions) and fine-grained ordering.
- `.text.unlikely` / `.text.hot`: hot/cold splitting for cache locality.
- `.data.rel.ro`: data that's logically const but contains pointers needing
  relocation (e.g. vtables under PIE) — placed so RELRO can re-protect it
  read-only after startup (Part 7.4).

**Try it ▸**

```bash
printf 'int a(){return 1;}int b(){return 2;}\n' > s.c
gcc -O2 -ffunction-sections -c s.c -o s.o
readelf -SW s.o | grep '.text'      # see .text.a and .text.b separately
```

---

## 3.2.4 Symbols: the assembler's contract with the linker

A **symbol** is a named value — usually an address (a section + offset), but
possibly an absolute constant or an external reference. Each symbol carries:

```
   Symbol record (conceptually):
   ┌──────────┬──────────┬───────────┬──────────┬──────────┬──────────┐
   │  name    │  value   │  size     │  type    │  binding │  section │
   │ "main"   │ offset   │ byte len  │ FUNC/    │ LOCAL/   │ .text /  │
   │          │          │           │ OBJECT/  │ GLOBAL/  │ UND/ABS/ │
   │          │          │           │ NOTYPE   │ WEAK     │ COMMON   │
   └──────────┴──────────┴───────────┴──────────┴──────────┴──────────┘
```

### Binding: who can see this symbol

```
   LOCAL   — visible only within this object file. The linker won't match it to
             other files. (e.g. `static` functions/vars, `.L` labels.)
   GLOBAL  — visible to all objects; participates in cross-file resolution.
             Exactly one object should *define* it; others reference it.
   WEAK    — like global, but: (a) doesn't cause a "multiple definition" error
             if a strong def exists (strong wins); (b) doesn't cause "undefined
             reference" if never defined (resolves to 0/NULL).
```

```
   .globl foo          → GLOBAL
   .weak  bar          → WEAK
   (no directive)      → LOCAL (for non-exported labels)
```

### Type

```
   FUNC    — a function (st_value = code address)
   OBJECT  — a data object (variable); has a size
   NOTYPE  — unspecified (plain label)
   SECTION — represents a section itself (used by relocations)
   FILE    — source file name (debugging breadcrumb)
   TLS     — thread-local storage symbol
```

### Special "sections" a symbol can belong to

```
   UND (SHN_UNDEF)  — undefined here; defined elsewhere. The linker MUST find a
                       definition (unless weak).  ← source of "undefined reference"
   ABS (SHN_ABS)    — an absolute value, not relative to any section.
   COMMON (SHN_COMMON) — a tentative definition (see §3.2.6).
```

---

## 3.2.5 Defined vs undefined: the linking handshake

```
   file a.o                          file b.o
   ────────                          ────────
   GLOBAL  foo  DEFINED in .text     GLOBAL  foo  UNDEFINED (UND)
        │                                 │
        └──────────── linker matches by name ───────────┘
                          ▼
              b.o's reference to foo is patched to point
              at a.o's definition (via a relocation).
```

- If `b.o` references `foo` but **no** object defines it → `undefined reference
  to 'foo'`.
- If **two** objects define the same GLOBAL `foo` → `multiple definition of
  'foo'` (unless one is weak).

This is *the* mechanism behind the two most common link errors. We expand it in
Part 5.2.

> **C++ ▸** `static` at namespace scope (or anonymous namespace) → LOCAL binding
> → invisible to other TUs → no clash. A normal `int g;` at namespace scope →
> GLOBAL → must be defined exactly once (ODR). `inline` functions/variables →
> emitted as **weak** COMDAT symbols so multiple TUs can each define them
> without a clash (Part 7.3).

---

## 3.2.6 Common symbols and tentative definitions (a C gotcha)

In C, a file-scope `int x;` *without* an initializer is a **tentative
definition**. Historically the compiler emitted it as a **COMMON** symbol, which
lets the linker *merge* multiple tentative definitions of the same name into one:

```
   a.c:  int x;        → COMMON x, size 4
   b.c:  int x;        → COMMON x, size 4
   link: ──────────────▶ one `x` in .bss   (NO multiple-definition error!)
```

This "common symbol" merging is a C-ism that has caused countless bugs (two
files accidentally sharing a global). Modern compilers default to
`-fno-common` (since GCC 10), making tentative definitions go straight to `.bss`
as regular symbols, so the duplicate now *correctly* errors.

```
   -fcommon  (old default):  duplicate `int x;` across files → silently merged
   -fno-common (new default): duplicate → multiple definition error  ✔ safer
```

> **C++ ▸** C++ never had tentative definitions — `int x;` at namespace scope is
> a *definition*, so this COMMON business is mostly a C concern. But you'll meet
> COMMON symbols when linking C libraries.

**Try it ▸**

```bash
printf 'int x;\n' > a.c; printf 'int x;\nint main(){return x;}\n' > b.c
gcc -fcommon    a.c b.c -o ok   2>&1 || true   # links (legacy)
gcc -fno-common a.c b.c -o bad  2>&1 || true   # multiple definition error
```

---

## 3.2.7 The string tables

Symbol *names* and section *names* aren't stored inline — they're offsets into
string tables (`.strtab` for symbols, `.shstrtab` for section names). A string
table is just NUL-separated strings; a name field is the byte offset of the
string's first character.

```
   .strtab bytes:   \0 m a i n \0 p r i n t f \0
   offset:          0  1       5  6           12
   symbol "main"  → st_name = 1
   symbol "printf"→ st_name = 6
```

This indirection keeps fixed-size symbol records small and lets identical names
share storage. We dissect it in Part 4.5.

---

## Summary

- A **section** is named bytes + flags (alloc/write/exec/tls/merge) + alignment;
  the assembler bins all output into sections.
- The alloc vs non-alloc split decides what's loaded at runtime vs only used at
  link/debug time (`.text` loads; `.symtab`/`.debug_*` don't).
- `.bss`/`.tbss` are NOBITS — size without file bytes.
- Fine-grained sections (`-ffunction-sections`, `.text.hot`, `.data.rel.ro`)
  enable GC and ordering by the linker.
- A **symbol** = name + value + size + type + binding + section. Binding
  (LOCAL/GLOBAL/WEAK) and the UND/ABS/COMMON pseudo-sections drive linking.
- Defined-vs-undefined matching is the mechanism behind "undefined reference"
  and "multiple definition"; COMMON symbols and `-fno-common` are a key C nuance.
- Names live in string tables, referenced by offset.

Next: [3.3 — Relocations: the assembler's promises](03-relocations.md)
