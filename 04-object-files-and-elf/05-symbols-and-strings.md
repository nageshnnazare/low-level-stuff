# 4.5 — Symbol Tables & String Tables

Symbols are how the linker and loader refer to functions and variables across
files. This chapter dissects the `Elf64_Sym` structure, the two symbol tables,
string tables, symbol versioning, and the GNU hash that makes runtime lookups
fast.

---

## 4.5.1 The `Elf64_Sym` structure

A symbol table is an array of these (24 bytes each, `sh_entsize = 24`):

```c
typedef struct {
    Elf64_Word    st_name;   /* 0  offset into the linked string table     */
    unsigned char st_info;   /* 4  binding (high 4 bits) + type (low 4)    */
    unsigned char st_other;  /* 5  visibility (low 2 bits)                 */
    Elf64_Half    st_shndx;  /* 6  section index, or SHN_UNDEF/ABS/COMMON  */
    Elf64_Addr    st_value;  /* 8  value: address, offset, or alignment    */
    Elf64_Xword   st_size;   /* 16 size in bytes (0 if unknown)            */
} Elf64_Sym;                 /* 24 bytes                                    */
```

```
   st_info byte:   ┌───────────┬───────────┐
                   │  binding  │   type    │
                   │ (4 bits)  │ (4 bits)  │
                   └───────────┴───────────┘
   binding = st_info >> 4          type = st_info & 0xF
   ELF64_ST_BIND(i) = (i)>>4       ELF64_ST_TYPE(i) = (i)&0xf
   ELF64_ST_INFO(b,t) = ((b)<<4)+((t)&0xf)
```

---

## 4.5.2 Binding, type, visibility decoded

### Binding (who can see / how it resolves)
```
   STB_LOCAL  (0)  visible only in this object; not matched across files
   STB_GLOBAL (1)  visible everywhere; participates in resolution
   STB_WEAK   (2)  global, but overridable; missing → 0, not an error
   STB_GNU_UNIQUE (10) GNU ext: one definition process-wide (for C++ inline
                       statics across dlopen'd libs)
```

### Type (what the symbol denotes)
```
   STT_NOTYPE  (0)  unspecified
   STT_OBJECT  (1)  a data object (variable) — has a size
   STT_FUNC    (2)  a function / executable code
   STT_SECTION (3)  a section (used as a relocation base; see §3.3.7)
   STT_FILE    (4)  source file name (st_shndx = SHN_ABS)
   STT_TLS     (6)  thread-local variable (st_value = offset in TLS block)
   STT_GNU_IFUNC(10)indirect function — resolved by calling a chooser at runtime
```

### Visibility (`st_other`, low 2 bits) — refines GLOBAL/WEAK
```
   STV_DEFAULT   (0) normal; can be preempted by other modules (interposition)
   STV_HIDDEN    (2) not exported from this module; usable internally only
   STV_PROTECTED (3) exported, but references within the module bind locally
   STV_INTERNAL  (1) like hidden, plus extra processor-specific constraints
```

> **C++ ▸** `-fvisibility=hidden` plus `__attribute__((visibility("default")))`
> (or the `[[gnu::visibility("default")]]`/export-macro idiom) is the standard
> way to control which symbols a shared library exposes. Hidden symbols don't go
> in `.dynsym`, shrinking the export table, speeding load, and enabling better
> inlining/optimization. This is the ELF analog of Windows' `__declspec(dllexport)`.

---

## 4.5.3 `st_value`, `st_shndx`, and `st_size` in context

The meaning of `st_value` depends on file type and `st_shndx`:

```
   In a relocatable .o:
     st_shndx = a real section index  → st_value = OFFSET within that section
     st_shndx = SHN_UNDEF             → undefined; st_value usually 0
     st_shndx = SHN_COMMON            → st_value = required ALIGNMENT, st_size=size
     st_shndx = SHN_ABS               → st_value = an absolute constant

   In an executable / shared object:
     st_value = the VIRTUAL ADDRESS of the symbol
```

```
   The linker's job (simplified): for each defined symbol, turn
   (section index, offset)  ──+ section's final base address──▶  virtual address
```

`st_size` is the object's byte size (a function's length, a variable's size). It
powers `nm --size-sort`, GDB's `p sizeof`, and bounds in sanitizers.

---

## 4.5.4 .symtab vs .dynsym: the two tables

```
 ┌───────────────────────┬───────────────────────┬────────────────────────────┐
 │                       │ .symtab (SHT_SYMTAB)  │ .dynsym (SHT_DYNSYM)       │
 ├───────────────────────┼───────────────────────┼────────────────────────────┤
 │ Purpose               │ link-time + debugging │ runtime dynamic linking    │
 │ Contains              │ ALL symbols (local +  │ only symbols needed at     │
 │                       │ global, incl. locals) │ runtime (exported/imported)│
 │ String table          │ .strtab               │ .dynstr                    │
 │ Loaded at runtime?    │ NO (not SHF_ALLOC)    │ YES (SHF_ALLOC)            │
 │ Survives strip?       │ NO (removed)          │ YES (required to run)      │
 │ Hash for fast lookup  │ none                  │ .gnu.hash / .hash          │
 └───────────────────────┴───────────────────────┴────────────────────────────┘
```

```
   FULL PICTURE
   ────────────
   .symtab ──names──▶ .strtab        (link-time; strippable; has locals)
   .dynsym ──names──▶ .dynstr        (run-time; mandatory; globals only)
         ▲
         └── .gnu.hash speeds "find symbol by name" for ld.so
```

This is why `nm a.out` on a stripped binary shows "no symbols" but `nm -D
a.out` (dynamic) still lists the exported/imported ones — `-D` reads `.dynsym`.

**Try it ▸**

```bash
nm demo        | head           # reads .symtab (gone if stripped)
nm -D demo     | head           # reads .dynsym (survives strip)
readelf -sW demo | head -20     # .dynsym then .symtab, with full fields
```

---

## 4.5.5 Reading symbol tables

```bash
readelf -sW demo
```

Annotated:

```
 Symbol table '.dynsym' contains N entries:
   Num: Value          Size Type    Bind   Vis      Ndx Name
     0: 0000...0000        0 NOTYPE  LOCAL  DEFAULT  UND          ← null symbol
     1: 0000...0000        0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.2.5
     2: 0000...0000        0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@...
        └ UND + GLIBC version → imported from libc                  ↑ versioned

 Symbol table '.symtab' contains M entries:
   ...
    52: 0000000000001149  47 FUNC    GLOBAL DEFAULT   13 main      ← defined in §13 (.text)
    48: 0000000000004010   4 OBJECT  GLOBAL DEFAULT   24 g_init    ← in .data
    49: 0000000000004014   4 OBJECT  GLOBAL DEFAULT   25 g_zero    ← in .bss
        │                  │                          │
        st_value (vaddr)   st_size                    st_shndx
```

Reading skills:
- `Ndx = UND` → undefined → must be resolved by the linker/loader.
- `Ndx = <n>` → defined in section *n* (cross-ref with `readelf -S`).
- `Bind = GLOBAL/WEAK/LOCAL`, `Type = FUNC/OBJECT/TLS`, `Vis = DEFAULT/HIDDEN`.
- `name@VERSION` → symbol versioning (next section).

---

## 4.5.6 String tables in detail

A string table (`SHT_STRTAB`) is a flat byte array of NUL-terminated strings.
Index 0 is always a single NUL so that `st_name = 0` means "no name".

```
   .strtab bytes:  00 6d 61 69 6e 00 67 5f 69 6e 69 74 00 70 72 69 6e 74 66 00
                   │  m  a  i  n  │  g  _  i  n  i  t  │  p  r  i  n  t  f  │
   offset:         0  1           5  6                12 13                 19

   symbol "main"   → st_name = 1
   symbol "g_init" → st_name = 6
   symbol "printf" → st_name = 13
   symbol with st_name = 0  → "" (anonymous)
```

Sub-string sharing is allowed: a name that's a suffix of another can point into
the middle. Linkers like `mold`/`lld` aggressively merge string tables.

**Try it ▸** Dump a string table raw:

```bash
readelf -p .strtab demo     # -p = print a string section
readelf -x .strtab demo | head   # -x = hex+ASCII of the section
```

---

## 4.5.7 Symbol versioning

GNU symbol versioning lets one shared library export multiple incompatible
versions of the same symbol (how glibc evolves without breaking old binaries):

```
   printf@GLIBC_2.2.5     ← default version a new link binds to
   memcpy@GLIBC_2.2.5     ← older
   memcpy@@GLIBC_2.14     ← @@ marks the DEFAULT version for new links
```

Three sections implement it:

```
   .gnu.version    (SHT_GNU_versym)  one 16-bit index per .dynsym entry
   .gnu.version_d  (SHT_GNU_verdef)  versions THIS library DEFINES
   .gnu.version_r  (SHT_GNU_verneed) versions this binary NEEDS from others
```

```
   binary ──needs──▶ printf@GLIBC_2.2.5
        .gnu.version_r:  "from libc.so.6, I need GLIBC_2.2.5"
   libc ──defines─▶ printf@@GLIBC_2.34, printf@GLIBC_2.2.5, ...
        .gnu.version_d:  the version tree it provides
```

> **War story ▸** The infamous `version 'GLIBC_2.34' not found` error: you built
> against a newer glibc (which bound your binary to `@GLIBC_2.34` versioned
> symbols) and tried to run it on a system with an older glibc that doesn't
> *define* that version. Versioning is doing exactly its job — refusing an
> incompatible match. Fixes: build on/against the older glibc, static-link, or
> use a sysroot.

**Try it ▸**

```bash
readelf -V demo          # all the version sections at once
objdump -T /lib/x86_64-linux-gnu/libc.so.6 | grep ' printf' | head
```

---

## 4.5.8 The GNU hash table (fast runtime lookup)

When `ld.so` resolves `printf`, it must find it among thousands of symbols in
libc. Linear search would be catastrophic, so `.dynsym` is accompanied by a hash
table. The modern `.gnu.hash` uses a **Bloom filter** front-end to reject
absent symbols in O(1) before probing buckets:

```
   lookup("printf"):
     1. h = gnu_hash("printf")
     2. Bloom filter test (a bitmask): if bit not set → DEFINITELY absent → skip
        (this is the big speedup: most "is it here?" queries fail fast)
     3. bucket = buckets[h % nbuckets]
     4. walk the chain comparing the stored hash, then strcmp on a hit
```

```
   .gnu.hash layout:
   ┌──────────┬───────────────┬──────────┬──────────────────────────────┐
   │ header   │ Bloom filter  │ buckets  │ chain (hash values, hi bit   │
   │ (counts) │ (bitmask)     │          │  marks end of a bucket chain)│
   └──────────┴───────────────┴──────────┴──────────────────────────────┘
```

The older `.hash` (SysV) is simpler (buckets + chains, no Bloom filter) and kept
only for compatibility. You rarely touch these by hand, but knowing they exist
explains why dynamic-symbol order in `.dynsym` is constrained (symbols are sorted
by bucket) and why huge export tables slow startup.

---

## Summary

- `Elf64_Sym` (24 bytes): name (strtab offset), `st_info` (binding+type),
  `st_other` (visibility), `st_shndx` (section/UND/ABS/COMMON), value, size.
- Binding (LOCAL/GLOBAL/WEAK), type (FUNC/OBJECT/TLS/IFUNC), and visibility
  (DEFAULT/HIDDEN/PROTECTED) fully characterize a symbol; visibility is the C++
  shared-library export knob.
- `.symtab`/`.strtab` are link-time + debug (strippable); `.dynsym`/`.dynstr`
  are the runtime subset (mandatory). `nm` vs `nm -D` read them respectively.
- String tables are NUL-separated, referenced by offset, with index 0 = "".
- Symbol **versioning** (`@`/`@@`, the `.gnu.version*` sections) lets libraries
  evolve; it's behind the classic `GLIBC_x.y not found` error.
- `.gnu.hash` (with a Bloom filter) makes runtime symbol lookup fast.

Next: [4.6 — Relocation entries on disk](06-relocations-on-disk.md)
