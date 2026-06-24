# 4.4 — Segments & the Program Header Table

Segments are the **execution view**: the coarse, page-aligned chunks the kernel
and dynamic loader actually map into memory. If sections are how the *linker*
thinks, segments are how the *loader* thinks. This chapter dissects the program
header table.

---

## 4.4.1 The `Elf64_Phdr` structure

The program header table is an array at `e_phoff`, with `e_phnum` entries each
`e_phentsize` (56) bytes:

```c
typedef struct {
    Elf64_Word  p_type;    /*  0  segment type (PT_*)                 */
    Elf64_Word  p_flags;   /*  4  permissions: PF_R|PF_W|PF_X         */
    Elf64_Off   p_offset;  /*  8  file offset of the segment's bytes  */
    Elf64_Addr  p_vaddr;   /* 16  virtual address to map to           */
    Elf64_Addr  p_paddr;   /* 24  physical addr (usually == p_vaddr)  */
    Elf64_Xword p_filesz;  /* 32  bytes present in the FILE           */
    Elf64_Xword p_memsz;   /* 40  bytes occupied in MEMORY            */
    Elf64_Xword p_align;   /* 48  alignment (page size for PT_LOAD)   */
} Elf64_Phdr;              /* 56 bytes                                 */
```

> **Field order quirk ▸** Unlike `Elf32_Phdr` (where `p_flags` comes last), the
> 64-bit version moves `p_flags` to offset 4 (right after `p_type`) for natural
> alignment of the 8-byte fields. A small detail that bites hand-parsers.

```
   p_offset ─▶ where in the FILE          p_filesz ─▶ bytes IN the file
   p_vaddr  ─▶ where in MEMORY            p_memsz  ─▶ bytes in memory
   p_flags  ─▶ R / W / X permissions      p_align  ─▶ alignment constraint
```

---

## 4.4.2 The critical `p_filesz` vs `p_memsz` distinction

These two differ precisely to implement `.bss`:

```
   p_filesz  =  bytes the loader copies from the file
   p_memsz   =  bytes the segment occupies in memory  (p_memsz >= p_filesz)
   The gap   =  p_memsz - p_filesz  →  ZERO-FILLED by the loader

   ┌───────────────────────────────────────────────────────────┐
   │  copied from file (p_filesz)   │  zero-filled (the rest)  │
   │  .data ...                     │  .bss ...                │
   └───────────────────────────────────────────────────────────┘
   │◀──────────────────── p_memsz ───────────────────────────▶│
   │◀───── p_filesz ──────▶│
```

This is the mechanism by which a 1-byte file can have a 1-GB zero-initialized
array: `.bss` contributes to `p_memsz` but not `p_filesz`.

---

## 4.4.3 Segment types (`p_type`)

```
 ┌──────────────────────┬────────────────────────────────────────────────────┐
 │ PT_NULL      (0)      │ unused entry                                      │
 │ PT_LOAD      (1)      │ a loadable chunk — THE important one              │
 │ PT_DYNAMIC   (2)      │ points to the .dynamic array (dynamic linking)    │
 │ PT_INTERP    (3)      │ path to the dynamic loader (e.g. ld-linux...)     │
 │ PT_NOTE      (4)      │ auxiliary notes (build-id, ABI)                   │
 │ PT_PHDR      (6)      │ describes the program header table itself         │
 │ PT_TLS       (7)      │ the thread-local storage template (Part 7.1)      │
 │ PT_GNU_EH_FRAME(0x6474e550)│ pointer to .eh_frame_hdr (unwinding index)   │
 │ PT_GNU_STACK (0x6474e551)│ stack permissions (PF_X here = exec stack!)    │
 │ PT_GNU_RELRO (0x6474e552)│ region to re-mark read-only after relocation   │
 │ PT_GNU_PROPERTY(0x6474e553)│ CET/BTI and other GNU properties             │
 └──────────────────────┴────────────────────────────────────────────────────┘
```

> **PT_GNU_STACK ▸** A program header with **no** `PF_X` flag tells the kernel
> the stack should be non-executable (W^X / NX). If this segment is missing or
> has `PF_X`, you get an *executable stack* — a security red flag. Linking an
> object that lacks `.note.GNU-stack` can silently make the whole binary's stack
> executable; modern linkers warn about this.

---

## 4.4.4 The canonical PT_LOAD layout

A dynamically-linked PIE typically has these `PT_LOAD` segments (modern linkers
emit up to four, splitting read-only/exec/RELRO/RW for tighter permissions):

```
 ┌────────┬───────┬───────────────────────────────────────────────────┐
 │ Segment│ Perms │ Contains (sections)                               │
 ├────────┼───────┼───────────────────────────────────────────────────┤
 │ LOAD 0 │ R     │ ELF hdr, phdrs, .interp, .note.*, .dynsym,        │
 │        │       │ .dynstr, .gnu.hash, .rela.*                       │
 │ LOAD 1 │ R E   │ .init .plt .text .fini                            │
 │ LOAD 2 │ R     │ .rodata .eh_frame_hdr .eh_frame                   │
 │ LOAD 3 │ R W   │ .init_array .fini_array .dynamic .got .got.plt    │
 │        │       │ .data .bss                                        │
 └────────┴───────┴───────────────────────────────────────────────────┘
```

Each `PT_LOAD` is **page-aligned** (`p_align = 0x1000` = 4 KiB on x86-64). The
loader `mmap`s each one with the requested permissions. Sections are grouped by
permission so each page has a single permission (enabling W^X enforcement).

```
   one PT_LOAD = one mmap() with fixed permissions
   ┌────────────────────────────────────────────────┐
   │ mmap(addr=p_vaddr+base, len=p_memsz,           │
   │      prot=PROT_READ|PROT_EXEC,                 │
   │      flags=MAP_PRIVATE|MAP_FIXED, fd, p_offset)│
   └────────────────────────────────────────────────┘
```

---

## 4.4.5 The other essential segments

### PT_INTERP — who runs first
Contains a path string like `/lib64/ld-linux-x86-64.so.2`. Its presence makes
the program **dynamically linked**: the kernel loads this interpreter and gives
*it* control, not your code. A *statically* linked executable has no PT_INTERP.

```bash
readelf -p .interp demo
#  [0]  /lib64/ld-linux-x86-64.so.2
```

### PT_DYNAMIC — the dynamic linking control block
Points to the `.dynamic` section: an array of tagged values (`Elf64_Dyn`) that
tells `ld.so` where the symbol table, relocations, needed libraries, init
functions, etc. are. We cover it in Part 5.3.

### PT_PHDR — self-reference
Describes the program header table itself, so the loader (and the program, via
`getauxval(AT_PHDR)`) can find the headers in memory. Used by `ld.so` to locate
its own and the executable's structures.

### PT_TLS — the thread-local template
Describes the initialization image for thread-local storage; each new thread
gets a fresh copy. Detailed in Part 7.1.

### PT_GNU_RELRO — partial read-only hardening
Marks a region (e.g. `.got`, `.dynamic`, `.init_array`) to be re-protected
**read-only after the dynamic loader finishes relocations**, closing GOT-overwrite
attack vectors. Part 7.4.

---

## 4.4.6 Section-to-segment mapping

The killer view that ties §4.3 and §4.4 together:

**Try it ▸**

```bash
readelf -l demo
```

```
 Program Headers:
   Type     Offset   VirtAddr  PhysAddr  FileSiz MemSiz  Flg Align
   PHDR     0x000040 0x000040  0x000040  0x0002d8 0x2d8   R   0x8
   INTERP   0x000318 0x000318  0x000318  0x00001c 0x1c    R   0x1
       [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
   LOAD     0x000000 0x000000  0x000000  0x000628 0x628   R   0x1000
   LOAD     0x001000 0x001000  0x001000  0x0001f5 0x1f5   R E 0x1000
   LOAD     0x002000 0x002000  0x002000  0x000114 0x114   R   0x1000
   LOAD     0x002db8 0x003db8  0x003db8  0x000260 0x268   RW  0x1000   ← memsz>filesz (.bss!)
   DYNAMIC  0x002dc8 0x003dc8  0x003dc8  0x0001f0 0x1f0   RW  0x8
   NOTE     ...
   GNU_EH_FRAME ...
   GNU_STACK 0x000000 0 0 0 0 RW  0x10     ← RW only, no X → non-exec stack ✔
   GNU_RELRO 0x002db8 0x003db8 ...         R   0x1

 Section to Segment mapping:
   Segment Sections...
   00
   01     .interp
   02     .interp .note.* .gnu.hash .dynsym .dynstr .gnu.version* .rela.dyn .rela.plt
   03     .init .plt .text .fini                       ← the R E (executable) LOAD
   04     .rodata .eh_frame_hdr .eh_frame
   05     .init_array .fini_array .dynamic .got .got.plt .data .bss   ← the RW LOAD
   ...
```

Notice in `LOAD` segment 05: `FileSiz 0x260 < MemSiz 0x268`. That 8-byte gap is
`.bss` — present in memory, absent from the file. And `GNU_STACK` is `RW` with no
`E`: the stack is non-executable.

---

## 4.4.7 Sections without segments, segments without unique sections

The two views overlap but aren't 1:1:

```
   Sections in NO segment:    .symtab .strtab .shstrtab .comment .debug_*
       → present in the file, never mapped to memory, strippable.

   Segments covering MANY sections:  one PT_LOAD ⊇ many sections.

   Segments covering NO file bytes:  PT_GNU_STACK (just a permission flag),
       parts of the last PT_LOAD beyond p_filesz (.bss).

   The SAME bytes in TWO segments:   e.g. the ELF header + phdrs are in the
       first PT_LOAD *and* described by PT_PHDR.
```

This is the practical face of §4.1.2's dual-nature diagram.

---

## 4.4.8 Why the kernel only needs program headers

When you `execve` a binary, the **kernel never reads the section header table**.
It reads the ELF header, then the program headers, maps the `PT_LOAD`s, finds
`PT_INTERP`, and transfers control. Sections are a *link-time* abstraction.

```
   execve path uses:  Ehdr → Phdrs → PT_LOAD mmaps → PT_INTERP → ld.so
   NEVER touches:     Shdrs, .symtab, .shstrtab
```

Proof: you can delete the section header table (`e_shoff = 0`) and the binary
still runs (though `objdump`/`gdb` get less helpful). Malware does this to resist
analysis; `strip --strip-all` and packers reduce sections aggressively.

**Try it ▸** Strip and observe it still runs but tools see less:

```bash
cp demo demo_stripped && strip --strip-all demo_stripped
./demo_stripped              # still runs
readelf -S demo_stripped | wc -l    # far fewer sections
readelf -l demo_stripped            # segments untouched — still loadable
```

---

## Summary

- The program header table (`e_phoff`, `e_phnum` × 56-byte `Elf64_Phdr`) is the
  execution view: page-aligned, permission-homogeneous segments.
- `p_filesz` < `p_memsz` implements `.bss` via zero-fill; `p_vaddr`/`p_offset`
  map memory↔file.
- `PT_LOAD` segments are `mmap`ed with `p_flags` permissions; modern binaries
  split them by permission for W^X.
- `PT_INTERP` names the dynamic loader; `PT_DYNAMIC`, `PT_TLS`, `PT_GNU_RELRO`,
  `PT_GNU_STACK` carry linking/security metadata.
- `readelf -l`'s "Section to Segment mapping" is the bridge between the linking
  and execution views.
- The kernel uses *only* program headers to run a binary; section headers are
  link/debug-time and strippable.

Next: [4.5 — Symbol tables & string tables](05-symbols-and-strings.md)
