# 2.1 — x86-64 Architecture for the Toolchain Hacker

You don't need to be an assembly *programmer* to master assemblers and linkers,
but you must be a fluent assembly *reader*. This chapter gives you exactly the
architectural model needed to read compiler output and disassembly.

---

## 2.1.1 What "x86-64" means

- **x86** = the 32-bit Intel instruction set lineage (8086 → 80386 → Pentium…).
- **x86-64** (a.k.a. **AMD64**, **Intel 64**, **EM64T**, **x64**) = the 64-bit
  extension AMD introduced in 2003. It adds 64-bit registers, 8 new GPRs
  (R8–R15), RIP-relative addressing, and a new 64-bit "long mode".
- The ELF machine constant is `EM_X86_64 = 62 (0x3E)`.

The CPU runs in **long mode** with two sub-modes: **64-bit mode** (what your
programs use) and **compatibility mode** (running old 32-bit binaries).

---

## 2.1.2 The register file (complete)

```
 Integer GPRs (64-bit, with sub-register names):
 ┌──────┬──────┬──────┬───────┐   ┌──────┬──────┬──────┬───────┐
 │ 64   │ 32   │ 16   │  8    │   │ 64   │ 32   │ 16   │  8    │
 ├──────┼──────┼──────┼───────┤   ├──────┼──────┼──────┼───────┤
 │ RAX  │ EAX  │ AX   │ AL/AH │   │ R8   │ R8D  │ R8W  │ R8B   │
 │ RBX  │ EBX  │ BX   │ BL/BH │   │ R9   │ R9D  │ R9W  │ R9B   │
 │ RCX  │ ECX  │ CX   │ CL/CH │   │ R10  │ R10D │ R10W │ R10B  │
 │ RDX  │ EDX  │ DX   │ DL/DH │   │ R11  │ R11D │ R11W │ R11B  │
 │ RSI  │ ESI  │ SI   │ SIL   │   │ R12  │ R12D │ R12W │ R12B  │
 │ RDI  │ EDI  │ DI   │ DIL   │   │ R13  │ R13D │ R13W │ R13B  │
 │ RBP  │ EBP  │ BP   │ BPL   │   │ R14  │ R14D │ R14W │ R14B  │
 │ RSP  │ ESP  │ SP   │ SPL   │   │ R15  │ R15D │ R15W │ R15B  │
 └──────┴──────┴──────┴───────┘   └──────┴──────┴──────┴───────┘

 Instruction pointer: RIP (64)     Flags: RFLAGS (64)

 Vector / FP registers:
   SSE:     XMM0 .. XMM15   (128-bit)
   AVX:     YMM0 .. YMM15   (256-bit, low half = XMM)
   AVX-512: ZMM0 .. ZMM31   (512-bit) + mask regs K0..K7

 Mostly-legacy: x87 FP stack ST(0)..ST(7), MMX MM0..MM7 (alias x87),
                segment regs CS/DS/SS/ES/FS/GS.
```

> **`AH`/`BH`/`CH`/`DH` ▸** The legacy high-byte registers can't be used in an
> instruction that also uses a REX-prefixed register. This encoding constraint
> occasionally shows up in disassembly and in why compilers prefer `SIL` over
> `DH`-style accesses.

---

## 2.1.3 CISC: memory operands in arithmetic

Unlike RISC, x86 lets a single instruction both compute *and* touch memory:

```asm
    add    rax, [rbx + rcx*4 + 8]   ; load from a computed address AND add, one insn
```

This is why x86 instructions are **variable length** (1 to 15 bytes) and why the
encoding (next chapter) is intricate. For reading disassembly, the key is the
**addressing mode** syntax inside brackets.

---

## 2.1.4 The addressing-mode formula

Every memory operand on x86-64 is computed by one general formula:

```
   effective_address = base + index * scale + displacement

   ┌──────┬──────────┬───────┬──────────────┐
   │ base │  index   │ scale │ displacement │
   │ reg  │  reg     │1/2/4/8│  signed imm  │
   └──────┴──────────┴───────┴──────────────┘
```

Intel syntax: `[base + index*scale + disp]`
AT&T syntax:  `disp(base, index, scale)`

```
 Examples (Intel / AT&T):
   [rax]                 movq (%rax), %rdx          ; *rax
   [rax + 8]             movq 8(%rax), %rdx         ; *(rax+8)   struct field
   [rax + rcx*4]         movq (%rax,%rcx,4), %rdx   ; array[i] for 4-byte elems
   [rax + rcx*8 + 16]    movq 16(%rax,%rcx,8), %rdx ; array of 8B + offset
   [rip + 0x2ed5]        movq 0x2ed5(%rip), %rdx    ; RIP-relative (PIC!)
```

> **The `scale*index` table indexing** is exactly C array subscripting:
> `arr[i]` for an `int arr[]` becomes `[arr_base + i*4]`. Recognizing this lets
> you reverse-engineer data structures from disassembly.

---

## 2.1.5 RIP-relative addressing: the linchpin of PIC

This deserves its own spotlight because it's central to position-independent
code, the GOT/PLT, and modern security.

In 32-bit x86 there was no way to address memory relative to the instruction
pointer; code that wanted its own address played tricks (`call $+5; pop`).
x86-64 added **RIP-relative addressing**: `[rip + disp32]` means "the address
`disp32` bytes past the *end* of this instruction."

```
   lea  rax, [rip + 0x2ed5]      ; rax = address_of_next_insn + 0x2ed5

   ┌─────────────── current instruction ────────────────┐
   │ 48 8d 05 d5 2e 00 00                               │
   └────────────────────────────────────────────────────┘
                                  ▲
                                  └ RIP points HERE (next insn) when disp is added
   target = RIP_of_next_insn + 0x00002ed5
```

Why it matters:

- Because the displacement is **relative**, the same code works no matter where
  the loader places the module → **position-independent code (PIC)** with zero
  runtime fixups for intra-module references.
- Accessing a global becomes `mov eax, [rip + off]` where `off` is filled in by
  the *linker* (a relocation). Accessing an *external* symbol or going through
  the GOT becomes `mov rax, [rip + off]` where the slot holds the real address
  filled in by the *dynamic loader*.

You'll see `R_X86_64_PC32`, `R_X86_64_GOTPCREL`, and `R_X86_64_PLT32`
relocations all tied to RIP-relative operands (Part 3.3 / 4.6).

---

## 2.1.6 A minimal instruction vocabulary

You can read 95% of compiler output with this set:

```
 DATA MOVEMENT
   mov   dst, src        copy
   movzx / movsx         copy with zero / sign extension
   lea   dst, [addr]     load *effective address* (computes, doesn't deref!)
   push / pop            stack
   xchg                  swap

 ARITHMETIC / LOGIC
   add sub imul idiv     integer math
   inc dec neg
   and or xor not        bitwise   (xor reg,reg = zero idiom)
   shl shr sar           shifts
   cmp                   subtract & set flags, discard result
   test                  AND & set flags, discard result

 CONTROL FLOW
   jmp                   unconditional
   je/jne jl/jg jb/ja…   conditional (read flags from prior cmp/test)
   call / ret            function call / return
   cmov<cc>              conditional move (branchless)

 SIMD / FP (common)
   movss/movsd           scalar float/double move
   addsd mulsd ...       scalar double math
   cvtsi2sd, cvttsd2si   int<->float conversions
```

> **`lea` is not a load ▸** `lea rax, [rbx+rcx*4+8]` computes `rbx+rcx*4+8`
> into `rax` *without touching memory*. Compilers love it as a fast 3-operand
> add/multiply. Mistaking it for a memory access is the #1 disassembly-reading
> error.

> **`xor eax, eax` ▸** The canonical way to zero a register — shorter than
> `mov eax, 0` and recognized by the CPU as a dependency-breaking idiom. You'll
> see it everywhere.

---

## 2.1.7 Reading a real function

Compile and disassemble a trivial function and map every line:

```c
int add3(int a, int b, int c) { return a + b + c; }
```

```asm
add3:
    lea    eax, [rdi + rsi]   ; eax = a + b      (rdi=a, rsi=b per ABI)
    add    eax, edx           ; eax = (a+b) + c  (edx=c)
    ret                       ; return value in eax
```

Everything here is ABI knowledge (args in `rdi,rsi,rdx`, return in `eax`) plus
the `lea`-as-add trick. That ABI is the subject of [2.4](04-calling-conventions-abi.md).

**Try it ▸**

```bash
printf 'int add3(int a,int b,int c){return a+b+c;}\n' > t.c
gcc -O2 -c t.c -o t.o && objdump -d t.o
```

---

## 2.1.8 Operating modes & the REX prefix (preview)

64-bit operations and access to R8–R15 require a **REX prefix** byte
(`0x40`–`0x4F`) prepended to the instruction. Its bits select 64-bit operand
size (`W`) and extend register fields (`R`, `X`, `B`).

```
   REX:  0100 W R X B
              │ │ │ └─ extends r/m or base field (to reach R8-R15)
              │ │ └─── extends index field
              │ └───── extends reg field
              └─────── 1 = 64-bit operand size
```

We fully decode instruction bytes in the next chapter. For now, just know: a
leading byte in `0x40..0x4F` is REX, and `0x48` (`W=1`) marks most 64-bit ops.

---

## Summary

- x86-64 is a little-endian CISC ISA with 16 GPRs, RIP, flags, and SIMD
  registers; variable-length instructions encode rich memory operands.
- The one addressing formula `base + index*scale + disp` covers all memory
  operands and directly mirrors C struct/array access.
- **RIP-relative addressing** is the foundation of position-independent code and
  the GOT/PLT machinery you'll meet in linking.
- A small instruction vocabulary plus the ABI lets you read almost all compiler
  output; `lea` and `xor reg,reg` are the two idioms to internalize first.

Next: [2.2 — Instructions, addressing modes & encoding](02-instructions-and-encoding.md)
