# 3.1 — How an Assembler Works (Two Passes)

The assembler is the simplest of the major toolchain stages, which makes it the
perfect place to understand the *mechanics* that the linker later relies on:
location counters, symbols, sections, and relocations. This chapter builds a
mental model of an assembler you could implement yourself.

---

## 3.1.1 The assembler's job in one sentence

> Translate textual assembly into a **relocatable object file**: machine-code
> bytes grouped into sections, a symbol table describing labels, and relocation
> records describing every reference whose final address isn't yet known.

Input: `.s` (text). Output: `.o` (ELF `ET_REL`). It does **not** resolve
references between files, assign final addresses, or know where the program will
live in memory. It produces *self-contained, relocatable* output with explicit
"to be filled in later" markers.

```
   foo.s  ──┐
            │  as (assembler)
            ▼
   ┌──────────────────────────────────────────────┐
   │ foo.o (ELF ET_REL)                           │
   │  ├─ .text       machine code                 │
   │  ├─ .data       initialized data             │
   │  ├─ .bss        (size only, no bytes)        │
   │  ├─ .rodata     constants                    │
   │  ├─ .symtab     symbols (labels)             │
   │  ├─ .strtab     symbol name strings          │
   │  ├─ .rela.text  relocations for .text        │
   │  └─ .rela.data  relocations for .data        │
   └──────────────────────────────────────────────┘
```

---

## 3.1.2 The location counter

The central piece of state in an assembler is the **location counter** (often
written `.` or `$`): the current offset within the *current section*. Every byte
emitted advances it. Every label captures its current value.

```
   Section .text, location counter starts at 0:

   offset  bytes        source
   ──────  ───────────  ────────────────
   0x00    55           push rbp           ; '.' was 0, now 1
   0x01    48 89 e5     mov rbp, rsp       ; '.' was 1, now 4
   0x04    b8 00 00 00 00   mov eax, 0     ; '.' was 4, now 9
   0x09    5d           pop rbp            ; '.' was 9, now 10
   0x0a    c3           ret                ; '.' was 10, now 11

   A label `main:` at the top captured '.' = 0x00 → main = .text+0
```

Each section has its *own* location counter. Switching sections (`.data` then
back to `.text`) resumes each counter where it left off.

```
   .text   . = 0 ──emit──▶ . = 0x20
   .data   . = 0 ──emit──▶ . = 0x08
   .text   . = 0x20 ──(resumes!)──▶ . = 0x33
```

The `.-main` in `.size main, .-main` is "current location counter minus the
value of `main`" = the byte length of the function. Now you know exactly what
that idiom computes.

---

## 3.1.3 Why two passes? The forward-reference problem

Consider:

```asm
        jmp   done        ; (1) forward reference — `done` not defined yet!
        nop
        nop
done:                     ; (2) now we learn done = current location
        ret
```

At line (1) the assembler must encode `jmp done`, but it doesn't yet know where
`done` is. The encoding of the jump's displacement depends on `done - here`.
This is the **forward reference problem**, and it's why classic assemblers make
**two passes**:

```
   ┌──────────────────── PASS 1 ──────────────────────┐
   │ Walk the source. Track the location counter.     │
   │ Record every label's (section, offset) in the    │
   │ symbol table. Determine each instruction's SIZE  │
   │ (so the counter is correct) but don't finalize   │
   │ operand values that depend on unknown symbols.   │
   └──────────────────────────────────────────────────┘
                         │  symbol table now complete
                         ▼
   ┌──────────────────── PASS 2 ──────────────────────┐
   │ Walk again. Now every label's value is known.    │
   │ Emit final bytes. For symbols whose addresses    │
   │ are STILL unknown (external, or needing link-    │
   │ time relocation), emit a RELOCATION record and   │
   │ a placeholder.                                   │
   └──────────────────────────────────────────────────┘
```

```
   Pass 1 sees:   jmp done   →  reserve 2 or 5 bytes? (the "branch sizing"
                                 / "span-dependent" problem — see §3.1.5)
   After pass 1:  done = 0x05
   Pass 2 emits:  eb 03      →  jmp rel8, displacement = done-(here+2) = 3
```

> **Internal vs external ▸** Pass 2 can fully resolve *internal* references
> (both label and use are in this file) by computing the offset itself — IF the
> reference is PC-relative within the same section. References that cross
> sections, or target external symbols, or need a GOT/PLT, become
> **relocations** left for the linker. (Even some same-file references emit
> relocations, because the assembler doesn't know the section's final address.)

---

## 3.1.4 A concrete two-pass trace

```asm
        .text
        .globl start
start:  call helper          ; A: forward ref to local label
        mov  eax, msglen      ; B: ref to an absolute symbol
        call printf           ; C: ref to EXTERNAL symbol
        ret
helper: ret

        .data
msg:    .ascii "hi"
        .set msglen, . - msg  ; D: msglen = 2 (computed at assemble time)
```

**Pass 1** (build symbol table):

```
   symbol   section  offset   notes
   ──────   ───────  ──────   ─────
   start    .text    0x00     global
   helper   .text    ????     local (resolved when reached)
   msg      .data    0x00     local
   msglen   ABS      0x02     absolute constant (. - msg)
   printf   UND      —        undefined (external) → linker must resolve
```

**Pass 2** (emit bytes + relocations):

```
   A: call helper   → helper is same-section, same file, PC-relative.
                      Assembler can compute displacement itself? In practice
                      GAS still emits R_X86_64_PLT32/PC32 for safety with
                      sections; for purely local it computes it. Either way the
                      *concept*: known distance → no reloc; unknown → reloc.
   B: mov eax,msglen→ msglen is ABS=2 → immediate 2 baked in, no reloc.
   C: call printf   → UND → emit `e8 00 00 00 00` + R_X86_64_PLT32(printf,-4).
```

This trace is the entire essence of assembly. Internalize: **known → bake it in;
unknown → leave a hole + a relocation.**

---

## 3.1.5 Branch sizing (span-dependent instructions)

A subtlety that makes "two passes" sometimes "two-and-a-bit". x86 jumps come in
short (`rel8`, 2 bytes) and near (`rel32`, 5/6 bytes) forms. The assembler wants
the short form when the target is close, but it can't know the distance until
addresses are assigned — and choosing short *changes* the addresses!

```
   jmp short:  eb XX            (2 bytes, range ±127)
   jmp near :  e9 XX XX XX XX   (5 bytes)

   If choosing short shrinks the code, later labels move closer, possibly
   letting other jumps also be short → iterative relaxation.
```

Assemblers solve this with **branch relaxation**: start optimistic (or
pessimistic) and iterate until sizes stabilize, or default to near and let the
linker *relax* later (the linker can also relax/shrink branches —
`-Wl,--relax`, important on RISC-V and ARM). This "span-dependent instruction"
problem is a classic compiler-construction topic.

---

## 3.1.6 What the assembler does NOT do

Crucial boundaries (these belong to the linker, Part 5):

- ✗ Resolve symbols across files.
- ✗ Assign final virtual addresses.
- ✗ Merge sections from multiple objects.
- ✗ Decide library/shared-object layout.
- ✗ Apply most relocations (it *creates* them; the linker *applies* them).

It *does*:

- ✓ Encode instructions to bytes.
- ✓ Maintain per-section location counters.
- ✓ Build the symbol table from labels/directives.
- ✓ Evaluate assemble-time-constant expressions.
- ✓ Emit relocation records for unresolved references.
- ✓ Lay out sections *within this one object*.

---

## 3.1.7 Modern reality: the integrated assembler

GCC writes textual `.s` and pipes it to a separate `as` process. **Clang/LLVM**,
by default, uses an **integrated assembler**: the compiler back end emits machine
code (an `MCStreamer` producing `MCInst`s) directly to an object file, skipping
the text round-trip for speed. The *conceptual* steps are identical; only the
plumbing differs.

```
   GCC:   cc1 ──.s──▶ as ──.o──▶
   Clang: clang back end ──(MC layer, in-process)──▶ .o    (no .s by default)
          (clang -S still emits .s for you to read)
```

LLVM's "MC" (Machine Code) layer is essentially a reusable assembler/disassembler
library; `llvm-mc` exposes it directly.

**Try it ▸**

```bash
llvm-mc -triple x86_64-linux-gnu -show-encoding <<< 'mov rax, rbx'
# shows: mov %rbx, %rax     # encoding: [0x48,0x89,0xd8]
echo '4889d8' | llvm-mc -triple x86_64-linux-gnu -disassemble -hex 2>/dev/null
```

---

## Summary

- The assembler turns text into a relocatable object: bytes in sections + a
  symbol table + relocations.
- The **location counter** (one per section) is the core state; labels capture
  it; `.-symbol` computes sizes.
- **Two passes** solve forward references: pass 1 fixes symbol values and
  instruction sizes; pass 2 emits bytes, leaving relocations for anything still
  unknown.
- **Branch relaxation** handles span-dependent jump sizing iteratively.
- The mantra: *known distance → bake it in; unknown → emit a hole + relocation.*
- Modern Clang uses an integrated (in-memory) assembler, but the model is the
  same.

Next: [3.2 — Sections, the location counter & symbols](02-sections-and-symbols.md)
