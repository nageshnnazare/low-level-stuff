# 3.3 — Relocations: The Assembler's Promises

A **relocation** is a note the assembler leaves for the linker (or loader)
saying: *"At this location, of this size, write the address of this symbol,
adjusted in this way."* Relocations are the glue of separate compilation. If you
deeply understand relocations, linking stops being magic.

---

## 3.3.1 The core idea

When the assembler encounters a reference whose final address it can't compute,
it does two things:

```
   1. Write a PLACEHOLDER in the instruction/data (usually 0).
   2. Emit a RELOCATION record describing how to fix the placeholder later.
```

```
   Source:    call printf
   Bytes:     e8 [00 00 00 00]          ← 4-byte hole at offset+1
                  └── placeholder
   Reloc:     { offset: +1,
                type:   R_X86_64_PLT32,
                symbol: printf,
                addend: -4 }
```

Later, the linker (or loader) reads each relocation and **patches** the bytes:

```
   patched_value = compute(type, S, A, P, ...)
   write patched_value into [offset .. offset+size]
```

---

## 3.3.2 The relocation equation and its variables

Every relocation type is a formula over a small set of values:

```
   S = the symbol's final value (its resolved address)
   A = the addend (a constant from the relocation or the original field)
   P = the place: the address of the storage being patched (Position)
   B = the base address of the shared object (for runtime relocs)
   G = the offset of the symbol's entry within the GOT
   GOT = the address of the Global Offset Table
   L = the address of the symbol's PLT entry
   Z = the size of the symbol
```

Common x86-64 types decode to these formulas:

```
 ┌────────────────────┬───────────────────────┬─────────────────────────────┐
 │ Type               │Formula (what to write)│ Used for                    │
 ├────────────────────┼───────────────────────┼─────────────────────────────┤
 │ R_X86_64_64        │ S + A                 │ 64-bit absolute address     │
 │ R_X86_64_32        │ S + A  (zero-ext)     │ 32-bit absolute (non-PIE)   │
 │ R_X86_64_32S       │ S + A  (sign-ext)     │ 32-bit absolute signed      │
 │ R_X86_64_PC32      │ S + A - P             │ 32-bit PC-relative (calls,  │
 │                    │                       │   RIP-relative data)        │
 │ R_X86_64_PLT32     │ L + A - P             │ call via PLT                │
 │ R_X86_64_GOTPCREL  │ G + GOT + A - P       │ RIP-rel load of GOT entry   │
 │ R_X86_64_GOTPCRELX │ (optimizable variant) │ GOT load, linker may relax  │
 │ R_X86_64_RELATIVE  │ B + A                 │ runtime: PIE/PIC base fixup │
 │ R_X86_64_GLOB_DAT  │ S                     │ runtime: fill a GOT slot    │
 │ R_X86_64_JUMP_SLOT │ S                     │ runtime: fill a PLT/GOT slot│
 │ R_X86_64_COPY      │ (copy Z bytes from S) │ runtime: copy reloc         │
 │ R_X86_64_TPOFF*/   │ TLS-model specific    │ thread-local storage        │
 │   DTPMOD/DTPOFF    │                       │   (Part 7.1)                │
 └────────────────────┴───────────────────────┴─────────────────────────────┘
```

> The genius of `R_X86_64_PC32` (`S + A - P`): it produces a **relative**
> displacement, so the patched code works regardless of where the module is
> loaded. The addend is usually `-4` because, for a RIP-relative operand, RIP
> points at the *end* of the instruction (4 bytes past the displacement field),
> so we subtract 4 to bias `P` to the right reference point.

---

## 3.3.3 Worked example: the addend `-4` explained

```
   call printf            occupying bytes at file offset 0x10..0x14
   ┌────┬────────────────┐
   │ e8 │ dd dd dd dd    │   dd = displacement field at offset 0x11
   └────┴────────────────┘
        ▲ P = 0x11 (the field being patched)
                        ▲ RIP at execution = 0x15 (end of instruction)

   We want:  displacement = target - RIP_at_exec = S - 0x15
   The formula gives:      S + A - P = S + A - 0x11
   For these to match:     A = -4    →  S - 4 - 0x11 = S - 0x15   ✓
```

So the `-4` addend compensates for the gap between the patched *field* (`P`) and
the *end of the instruction* (where RIP actually is). This is why nearly every
x86-64 PC-relative relocation has addend `-4` (or `-1`, `-2` for shorter
fields).

---

## 3.3.4 REL vs RELA: where the addend lives

ELF has two relocation record formats:

```
   Elf64_Rel  { r_offset; r_info; }              ← addend stored IN the field
   Elf64_Rela { r_offset; r_info; r_addend; }    ← addend stored in the record
```

```
   r_offset : where to patch (offset in the target section / vaddr)
   r_info   : packs the symbol index (high 32 bits) and type (low 32 bits)
              sym  = r_info >> 32
              type = r_info & 0xffffffff
   r_addend : (RELA only) the constant A
```

- **x86-64 uses RELA** (`.rela.text`, `.rela.data`) — addend in the record,
  field starts as zero. Cleaner.
- **i386 (32-bit) uses REL** (`.rel.text`) — addend is stored *in* the field
  being patched (the "implicit addend"). The linker reads the field, computes,
  writes back.
- ARM mixes both depending on context.

```
   RELA:   field = 0 ;  record carries A          → patched = f(S, A_record)
   REL:    field = A ;  record carries no addend  → patched = f(S, field_value)
```

---

## 3.3.5 Reading real relocations

**Try it ▸**

```bash
cat > r.c <<'EOF'
extern int ext_var;
int  global_var = 7;
int *p = &global_var;          // needs absolute reloc to global_var
int read_ext(void){ return ext_var; }   // needs reloc to external symbol
EOF
gcc -c -fno-pic r.c -o r.o      # -fno-pic to see classic absolute relocs
readelf -rW r.o
objdump -dr r.o
```

You'll see something like:

```
 Relocation section '.rela.data' ...
   Offset          Info           Type           Sym. Value   Sym. Name + Addend
   0000000000000000 0007 R_X86_64_64  0000000000000000 global_var + 0
       └ at .data offset 0 (where `p` lives), write address of global_var

 Relocation section '.rela.text' ...
   ...              R_X86_64_PC32 ext_var - 4
       └ in read_ext's code, RIP-relative load of ext_var
```

Now the disassembly's `mov eax, [rip+0x0]  # ext_var` makes sense: the `0x0` is
a placeholder, and the relocation tells the linker to compute the real
RIP-relative displacement.

---

## 3.3.6 Link-time vs load-time relocations

A vital distinction:

```
   STATIC (link-time) relocations          DYNAMIC (load-time) relocations
   ──────────────────────────────         ───────────────────────────────
   In .rela.text / .rela.data              In .rela.dyn / .rela.plt
   Consumed by the LINKER                  Consumed by the dynamic LOADER (ld.so)
   Fully resolved → vanish from output     Survive into the executable/.so
   e.g. R_X86_64_PC32, R_X86_64_PLT32      e.g. R_X86_64_RELATIVE, GLOB_DAT,
                                                JUMP_SLOT, COPY
```

```
   .o files         linker          executable / .so          process
   ────────         ──────          ────────────────          ───────
   .rela.text  ──▶  applied   ──▶   (gone)
   .rela.data  ──▶  applied   ──▶   (gone)
                                    .rela.dyn   ──▶ ld.so ──▶ patched at load
                                    .rela.plt   ──▶ ld.so ──▶ patched (lazy/now)
```

Why some relocations *survive* to runtime: in a position-independent
executable/shared library, the load address isn't known until the loader picks
one (ASLR). So absolute pointers (vtables, `int *p = &g;`, GOT entries) must be
fixed up *at load time* by the dynamic loader. These become `R_X86_64_RELATIVE`
(add load base) or `GLOB_DAT`/`JUMP_SLOT` (resolve external symbols). This is the
bridge to Parts 5.3–5.4.

> **Performance note ▸** A binary with millions of `R_X86_64_RELATIVE`
> relocations (large C++ apps full of vtables and pointers) pays a startup cost
> as ld.so walks them all. `DT_RELR` (relative relocation compression, ~2020)
> shrinks this dramatically and is now used by Android and modern glibc.

---

## 3.3.7 Section symbols and the `+ addend` trick

Relocations often reference a **section symbol** plus an addend rather than a
named symbol. For local targets, the assembler may emit `.text + 0x40` instead
of `helper`:

```
   R_X86_64_PC32   .text + 0x3c        (reference to a local label at .text+0x40,
                                         with the usual -4 bias folded in)
```

This is why `readelf -r` sometimes shows section names (`.text`, `.rodata`) with
big addends instead of the function/variable name — the symbol is the *section*,
and the addend selects the offset within it. Local symbols don't need to be
individually exported, so the assembler economizes by using the section symbol.

---

## 3.3.8 Relocation overflow: a real failure mode

If a relocation's computed value doesn't fit the field, you get an error:

```
   relocation truncated to fit: R_X86_64_PC32 against symbol `foo'
```

This happens when, e.g., code and data end up more than ±2 GiB apart so a 32-bit
PC-relative displacement can't reach. Causes and fixes:

- Huge binaries / large data → compile with `-mcmodel=large` or `medium`
  (uses 64-bit addressing for big objects).
- Calling a far symbol via a `rel32` `call` → the linker inserts a **thunk/stub**
  (a "veneer" on ARM) that does an indirect jump, bridging the distance.

```
   Code model controls the assumptions:
     -mcmodel=small  : code+data within low 2 GiB (default). 32-bit relocs OK.
     -mcmodel=medium : large data, small code.
     -mcmodel=large  : no size assumptions; 64-bit addressing everywhere.
```

---

## Summary

- A relocation = (offset, type, symbol, addend): write `f(S,A,P,…)` into the
  hole.
- Each type is a tiny formula; `S+A` (absolute), `S+A-P` (PC-relative),
  `L+A-P` (PLT), `B+A` (load-base) are the workhorses. The ubiquitous `-4`
  addend compensates for RIP pointing at the instruction's end.
- **RELA** (x86-64) stores the addend in the record; **REL** (i386) stores it in
  the field.
- Link-time relocations are applied by the linker and vanish; **dynamic**
  relocations survive into the binary and are applied by `ld.so` at load (and are
  why PIE/shared libs work under ASLR).
- Section-symbol-plus-addend, relocation overflow, code models, and DT_RELR
  compression are the practical wrinkles experts hit.

This completes Part 3. → Part 4 opens up the container that holds all of this:
[4.1 — ELF overview & the dual nature of the format](../04-object-files-and-elf/01-elf-overview.md)
