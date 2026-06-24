# 2.2 — Instruction Encoding: From Mnemonic to Bytes

This is what an assembler *does* at its core: turn `mov rax, 5` into bytes. To
understand assemblers, relocations, and disassembly, you must understand x86-64
instruction encoding. We'll decode real instructions byte by byte.

> If you only ever target ARM/RISC-V (fixed 32-bit instructions) this chapter is
> easier there — but x86 encoding is the gold standard for "why is this hard?"
> and the concepts (opcode, operands, immediates, displacements) are universal.

---

## 2.2.1 The anatomy of an x86-64 instruction

An x86-64 instruction is a sequence of up to 15 bytes in this fixed *order* of
optional components:

```
 ┌──────────┬──────┬────────┬────────┬──────┬─────┬──────────────┬───────────┐
 │ Legacy   │ REX  │ Opcode │ ModR/M │ SIB  │Disp │  Immediate   │           │
 │ prefixes │prefix│ 1-3 B  │  1 B   │ 1 B  │1/2/4│   1/2/4/8 B  │           │
 │ 0-4 B    │ 0-1B │        │ 0-1 B  │ 0-1B │  B  │              │           │
 └──────────┴──────┴────────┴────────┴──────┴─────┴──────────────┴───────────┘
   optional   opt    REQUIRED  opt      opt    opt      opt

 Total length ≤ 15 bytes (CPU enforces this).
```

| Field | Purpose |
|-------|---------|
| **Legacy prefixes** | operand-size (`0x66`), address-size (`0x67`), `lock`, `rep`/`repne`, segment overrides (`0x64`=FS, `0x65`=GS), branch hints |
| **REX** | 64-bit operand size + register extension (x86-64 only) |
| **Opcode** | the operation; 1–3 bytes (the `0x0F` escape introduces 2-byte opcodes) |
| **ModR/M** | encodes operands: a register and a register-or-memory operand |
| **SIB** | "Scale-Index-Base" — extends ModR/M for complex addressing |
| **Displacement** | constant offset for memory operands |
| **Immediate** | a constant operand baked into the instruction |

---

## 2.2.2 The ModR/M byte

The ModR/M byte is the heart of operand encoding:

```
   7   6 5     3 2     0
 ┌─────┬───────┬───────┐
 │ mod │  reg  │  r/m  │
 │ 2b  │  3b   │  3b   │
 └─────┴───────┴───────┘

 mod = 11  → r/m is a REGISTER
 mod = 00  → memory, [reg]            (no displacement)*
 mod = 01  → memory, [reg + disp8]    (1-byte signed displacement)
 mod = 10  → memory, [reg + disp32]   (4-byte signed displacement)

 reg = a register operand (or an opcode extension for some instructions)
 r/m = a register or the base of a memory operand
       (r/m == 100 means "an SIB byte follows")
       (r/m == 101 with mod==00 means RIP-relative [rip + disp32]!)
```

The 3-bit `reg`/`r/m` fields select registers 0–7; REX.R / REX.B / REX.X extend
them to 8–15.

```
 Register number encoding (low 3 bits; +8 if REX bit set):
   0=A/AX/EAX/RAX   1=C/CX   2=D/DX   3=B/BX
   4=SP             5=BP     6=SI     7=DI
   (with REX.B/R/X) 8=R8 ... 15=R15
```

---

## 2.2.3 Worked example: decode `48 89 d8`

```
 Bytes:  48        89        d8
         REX.W     opcode    ModR/M

 48 = 0100 1000   → REX with W=1 (64-bit), R=0, X=0, B=0
 89 = opcode for  MOV r/m64, r64   (store the reg operand into r/m)
 d8 = 1101 1000   → mod=11 (r/m is a register)
                     reg = 011 = 3 = RBX (source)
                     r/m = 000 = 0 = RAX (dest)

 Decoded:  mov rax, rbx     (AT&T: movq %rbx, %rax)
```

> Note opcode `0x89` is "MOV *to* r/m from reg", so `reg` is the **source** and
> `r/m` the **destination**. Opcode `0x8B` is the reverse direction. Direction
> bits like this are why hand-decoding requires an opcode table.

## 2.2.4 Worked example: a memory operand with SIB

Decode `mov eax, [rbx + rcx*4 + 0x10]`:

```
 Bytes:  8b   44   8b   10
         │    │    │    └─ disp8 = 0x10
         │    │    └────── SIB byte
         │    └─────────── ModR/M
         └──────────────── opcode (MOV r32, r/m32)

 ModR/M 0x44 = 01 000 100
   mod=01 → memory + disp8
   reg=000 → EAX (destination)
   r/m=100 → SIB follows

 SIB 0x8b = 10 001 011
   scale=10 → *4
   index=001 → ECX/RCX
   base =011 → EBX/RBX

 → mov eax, [rbx + rcx*4 + 0x10]
```

The SIB byte mirrors the addressing formula exactly:

```
   7   6 5      3 2     0
 ┌─────┬────────┬───────┐
 │scale│ index  │ base  │     ea = base + index*(2^scale) + disp
 │ 2b  │  3b    │  3b   │
 └─────┴────────┴───────┘
   00=*1 01=*2 10=*4 11=*8
```

---

## 2.2.5 Immediates and displacements — where relocations live

This is the bridge to Part 3. An **immediate** is a constant operand; a
**displacement** is a constant address offset. When that constant is *the
address of a symbol the assembler doesn't yet know*, the assembler:

1. Emits the instruction with a **placeholder** (usually zeros) in the
   immediate/displacement field.
2. Emits a **relocation record** that says "at this byte offset, of this size,
   patch in the address of symbol X (possibly with an addend / relative
   adjustment)."

```
   call printf            ; printf is in libc, address unknown at assemble time

   Encoded:  e8 00 00 00 00
             │  └────┬────┘
             │       └ disp32 placeholder = 0  ← THE HOLE
             └ opcode E8 = CALL rel32

   Relocation: offset=+1, type=R_X86_64_PLT32, symbol=printf, addend=-4
               "patch these 4 bytes with (printf_PLT - (here+4))"
```

So **the displacement/immediate field is literally the hole that relocation
fills.** When you see `objdump -d` print `call <printf>` next to `e8 00 00 00
00`, the `<printf>` is `objdump` reading the relocation, not the bytes.

**Try it ▸** See the hole and its relocation together:

```bash
printf '#include <cstdio>\nint main(){puts("hi");}\n' > t.cpp
g++ -c t.cpp -o t.o
objdump -dr t.o            # -r interleaves relocations with disassembly
# You'll see:  e8 00 00 00 00   call ...   <reloc: R_X86_64_PLT32 puts-0x4>
```

---

## 2.2.6 Instruction length: why x86 disassembly can desync

Because instructions vary 1–15 bytes and the length depends on prefixes,
ModR/M, SIB, and immediates, a disassembler must decode from a known starting
point. Starting one byte off produces garbage that *eventually* resynchronizes
(self-synchronizing property) but can hide instructions — a fact exploited by
obfuscators and exploited *against* by malware analysts.

```
 Correct alignment:    [48 89 d8][c3]               mov rax,rbx ; ret
 Off-by-one (wrong):   [89 d8][c3]...               mov eax,ebx ; ret  (different!)
```

This is why `objdump` uses the symbol table and relocations to find function
boundaries, and why linear-sweep vs recursive-descent disassembly differ.

---

## 2.2.7 ARM64 / RISC-V: fixed-width sanity

For contrast, AArch64 instructions are **all exactly 4 bytes**, little-endian,
with fields packed at fixed bit positions:

```
 AArch64 ADD (immediate):  ADD Xd, Xn, #imm
  31         24 23 22 21      10 9   5 4    0
 ┌───────────┬──┬─────────────┬──────┬──────┐
 │ 1001000100│sh│   imm12     │  Rn  │  Rd  │
 └───────────┴──┴─────────────┴──────┴──────┘
```

Decoding is trivial by comparison — but the *immediate* still can't hold a full
64-bit address, so ARM needs `ADRP` + `ADD`/`LDR` *pairs* and corresponding
relocation types (`R_AARCH64_ADR_PREL_PG_HI21`, `R_AARCH64_ADD_ABS_LO12_NC`).
The relocation *concept* is identical; only the granularity differs.

---

## 2.2.8 Tools for encoding work

| Task | Tool |
|------|------|
| Assemble one instruction to bytes | `echo 'mov rax, rbx' | as -o /dev/null` (or use `rasm2 'mov rax, rbx'` from radare2) |
| Disassemble raw bytes | `rasm2 -d '4889d8'` or `objdump -D -b binary -m i386:x86-64` |
| Full encoding reference | Intel SDM Vol. 2, or the [felixcloutier.com](https://www.felixcloutier.com/x86/) tables |
| Quick byte-level disasm | `objdump -d --insn-width=15` shows raw bytes per insn |

**Try it ▸** Round-trip an instruction:

```bash
# requires radare2: brew install radare2  /  apt install radare2
rasm2 -a x86 -b 64 'mov rax, rbx'      # -> 4889d8
rasm2 -a x86 -b 64 -d '4889d8'         # -> mov rax, rbx
```

---

## Summary

- An x86-64 instruction = optional prefixes + REX + opcode + ModR/M + SIB +
  displacement + immediate, in that order, ≤ 15 bytes.
- The **ModR/M** and **SIB** bytes encode operands and mirror the
  `base+index*scale+disp` addressing formula exactly.
- **Immediates and displacements are the holes that relocations fill** — this is
  the direct link from encoding to linking.
- Variable length makes disassembly position-sensitive; fixed-width ISAs (ARM,
  RISC-V) trade that for needing instruction *pairs* to build large constants,
  which is why their relocation sets are larger.

Next: [2.3 — Assembly syntax, directives & GAS vs NASM](03-syntax-and-directives.md)
