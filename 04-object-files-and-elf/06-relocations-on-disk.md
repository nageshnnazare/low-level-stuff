# 4.6 — Relocation Entries on Disk

Part 3.3 explained relocations *conceptually*. Now we pin them to their exact ELF
on-disk representation and walk a complete patch by hand, then connect the
link-time and dynamic relocation sections.

---

## 4.6.1 The `Elf64_Rela` / `Elf64_Rel` structures

```c
typedef struct {                 typedef struct {
    Elf64_Addr   r_offset;           Elf64_Addr  r_offset;
    Elf64_Xword  r_info;             Elf64_Xword r_info;
    Elf64_Sxword r_addend;       } Elf64_Rel;     /* no addend */
} Elf64_Rela;                    /* x86-64 uses RELA exclusively */
```

```
   r_offset : WHERE to patch.
              In a .o: offset within the target section.
              In an exe/.so: the virtual address to patch.
   r_info   : packs symbol index + relocation type.
              sym  = r_info >> 32          (ELF64_R_SYM)
              type = r_info & 0xffffffff   (ELF64_R_TYPE)
   r_addend : the constant A in the relocation formula (RELA only).
```

```
   r_info (64 bits):
   ┌────────────────────────────┬────────────────────────────┐
   │   symbol index (32 bits)   │  relocation type (32 bits) │
   └────────────────────────────┴────────────────────────────┘
```

The symbol index points into the symbol table named by the relocation section's
`sh_link` (`.symtab` for `.rela.text`, `.dynsym` for `.rela.dyn`).

---

## 4.6.2 Which section holds which relocations

```
 ┌──────────────┬──────────┬─────────┬────────────────────────────────────────┐
 │ Section      │ sh_link  │ sh_info │ Consumed by / purpose                  │
 ├──────────────┼──────────┼─────────┼────────────────────────────────────────┤
 │ .rela.text   │ .symtab  │ .text   │ LINKER: patch code references          │
 │ .rela.data   │ .symtab  │ .data   │ LINKER: patch data (e.g. int *p=&g)    │
 │ .rela.eh_frame│ .symtab │ .eh_fr. │ LINKER: patch unwind tables            │
 │ .rela.dyn    │ .dynsym  │ —       │ LOADER: data relocs at load time       │
 │ .rela.plt    │ .dynsym  │ .got.plt│ LOADER: PLT/GOT slots (lazy or now)    │
 └──────────────┴──────────┴─────────┴────────────────────────────────────────┘
```

```
   LINK TIME                              LOAD TIME
   ─────────                              ─────────
   .rela.text ─┐                          .rela.dyn ─┐
   .rela.data ─┼─▶ ld applies & DELETES   .rela.plt ─┴─▶ ld.so applies at startup
   .rela.*    ─┘   (gone from output)         (these SURVIVE into the binary)
```

The `.o` has `.rela.text`/`.rela.data` (linker-consumed); the final
executable/`.so` has `.rela.dyn`/`.rela.plt` (loader-consumed). This split is the
on-disk realization of §3.3.6.

---

## 4.6.3 Hand-applying a relocation, end to end

Let's trace one `R_X86_64_PC32` from `.o` to patched executable.

**Step 1 — the source and object:**

```c
extern int counter;
int read_counter(void){ return counter; }
```

```bash
gcc -O2 -c rc.c -o rc.o
objdump -dr rc.o
```

```
 0000000000000000 <read_counter>:
    0:  8b 05 00 00 00 00     mov    eax, DWORD PTR [rip+0x0]    # 6 <read_counter+0x6>
                  ▲                                        2: R_X86_64_PC32  counter-0x4
                  └ the 4-byte disp32 hole at offset 0x2
    6:  c3                    ret
```

**Step 2 — read the relocation record:**

```bash
readelf -rW rc.o
# Offset  Info             Type           Sym. Name + Addend
# 00...02 000X00000002     R_X86_64_PC32  counter - 4
```

```
   r_offset = 0x2          ← patch the 4 bytes at .text+0x2
   type     = R_X86_64_PC32 → write  S + A - P
   sym      = counter
   r_addend = -4           ← compensates for RIP = end of instruction
```

**Step 3 — the linker assigns addresses (hypothetical):**

```
   read_counter (.text) placed at vaddr 0x1130  → the insn at 0x1130
   the disp32 field P  = 0x1130 + 0x2 = 0x1132
   counter (.bss) placed at vaddr 0x4040 → S = 0x4040
```

**Step 4 — compute and patch:**

```
   value = S + A - P
         = 0x4040 + (-4) - 0x1132
         = 0x4040 - 0x1136
         = 0x2F0A          (a 32-bit signed displacement)

   Write 0x00002F0A little-endian into the hole:
   before:  8b 05 [00 00 00 00]
   after:   8b 05 [0a 2f 00 00]
```

**Step 5 — verify the result in the linked binary:**

```bash
gcc rc.o other_defining_counter.o -o rc && objdump -d rc | grep -A1 read_counter
# mov eax, DWORD PTR [rip+0x2f0a]   # 4040 <counter>
```

At runtime the CPU computes `RIP(0x...1136) + 0x2F0A = 0x...4040` and loads
`counter`. The relocation has done its job: a position-independent reference.

> This single worked example *is* linking in miniature. Every relocation the
> linker applies is this same five-step dance, just at scale.

---

## 4.6.4 Dynamic relocations: the runtime variants

In the final binary, references that can't be fully resolved at link time become
dynamic relocations applied by `ld.so`. The common x86-64 ones:

```
 ┌────────────────────┬───────────────────────────────────────────────────────┐
 │ R_X86_64_RELATIVE  │ B + A. No symbol — just "add the load base to A".     │
 │                    │ Used for internal absolute pointers in PIE/PIC.       │
 │                    │ The bulk of relocations in a big C++ binary.          │
 │ R_X86_64_GLOB_DAT  │ S. Write a resolved symbol's address into a GOT slot. │
 │                    │ Used for data imported from another module.           │
 │ R_X86_64_JUMP_SLOT │ S. Write a function's address into a .got.plt slot.   │
 │                    │ Applied lazily (on first call) or eagerly (BIND_NOW). │
 │ R_X86_64_COPY      │ Copy Z bytes of an exported variable into the exe's   │
 │                    │ .bss (copy relocation; legacy data-import mechanism). │
 │ R_X86_64_IRELATIVE │ Call an IFUNC resolver, store its result. CPU-feature │
 │                    │ dispatch (e.g. picking an AVX memcpy at startup).     │
 └────────────────────┴───────────────────────────────────────────────────────┘
```

```
   ld.so startup loop (conceptually):
     for each reloc in .rela.dyn:
        switch type:
          RELATIVE : *(addr) = load_base + addend
          GLOB_DAT : *(addr) = resolve(symbol)
          COPY     : memcpy(addr, resolve(symbol), size)
          IRELATIVE: *(addr) = ((resolver_fn)(load_base+addend))()
     for each reloc in .rela.plt:   (if BIND_NOW; else done lazily)
          JUMP_SLOT: *(got_slot) = resolve(symbol)
```

**Try it ▸**

```bash
readelf -rW demo            # see .rela.dyn (RELATIVE/GLOB_DAT) and .rela.plt (JUMP_SLOT)
readelf -d demo | grep -E 'RELA|JMPREL|PLTREL|FLAGS'   # the .dynamic tags that drive it
```

---

## 4.6.5 R_X86_64_RELATIVE and why PIE startup costs add up

In a PIE, every absolute pointer baked into writable data needs a `RELATIVE`
fixup at load time (because the base is randomized). Consider:

```c
int  arr[3] = {1,2,3};
int *table[] = { &arr[0], &arr[1], &arr[2] };   // 3 pointers → 3 RELATIVE relocs
```

A large C++ program with thousands of vtables (each full of function pointers)
and pointer tables can have **hundreds of thousands** of `R_X86_64_RELATIVE`
relocations, each processed at startup.

```
   each RELATIVE reloc at load:   *(base + r_offset) += base
   (a memory write per pointer — measurable on large binaries)
```

Mitigations:
- **`DT_RELR`** / `.relr.dyn`: a compressed encoding for the (overwhelmingly
  common) relative relocations, replacing 24 bytes per reloc with a bitmap. Huge
  win; used by Android and recent glibc/lld. Enable with
  `-Wl,-z,pack-relative-relocs`.
- **`-fno-pic`/`-no-pie`**: if you don't need ASLR, fixed addresses avoid the
  relocations entirely (at a security cost).

---

## 4.6.6 The `.dynamic` array ties it together (preview of Part 5)

`ld.so` finds all these relocation sections via the `.dynamic` array
(`PT_DYNAMIC`), a list of `(tag, value)` pairs:

```c
typedef struct { Elf64_Sxword d_tag; union { Elf64_Xword d_val; Elf64_Addr d_ptr; } d_un; } Elf64_Dyn;
```

```
   DT_RELA      → address of .rela.dyn
   DT_RELASZ    → its total size
   DT_RELAENT   → size of one entry (24)
   DT_JMPREL    → address of .rela.plt
   DT_PLTRELSZ  → its size
   DT_PLTREL    → which type (DT_RELA)
   DT_SYMTAB    → address of .dynsym
   DT_STRTAB    → address of .dynstr
   DT_RELR/DT_RELRSZ → compressed relative relocations
   DT_FLAGS     → BIND_NOW etc.
```

So at startup, `ld.so` reads `.dynamic`, finds `DT_RELA`/`DT_RELASZ`, and loops
over the entries exactly as in §4.6.4. We cover `.dynamic` fully in Part 5.3.

---

## Summary

- On disk, a relocation is `r_offset` (where), `r_info` (symbol index + type),
  and (RELA) `r_addend` (the constant A). `sym = r_info>>32`,
  `type = r_info & 0xffffffff`.
- Relocation sections split into link-time (`.rela.text`/`.rela.data`,
  consumed and removed by `ld`) and load-time (`.rela.dyn`/`.rela.plt`, consumed
  by `ld.so`), wired up via `sh_link`/`sh_info` and the `.dynamic` tags.
- Applying a relocation is a deterministic five-step computation; the
  `R_X86_64_PC32` walk-through is linking in miniature.
- Dynamic relocations (`RELATIVE`, `GLOB_DAT`, `JUMP_SLOT`, `COPY`, `IRELATIVE`)
  are applied by the loader; `RELATIVE` dominates in PIE binaries and motivates
  `DT_RELR` compression.

This completes Part 4. → Part 5 puts it all in motion:
[5.1 — Static linking from start to finish](../05-linking/01-static-linking.md)
