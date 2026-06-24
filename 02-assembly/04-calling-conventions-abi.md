# 2.4 — Calling Conventions & the ABI

An **ABI** (Application Binary Interface) is the contract that lets separately
compiled pieces of code interoperate: how arguments are passed, who saves which
registers, how the stack is laid out, how structs are returned, how names are
mangled, how exceptions unwind. **The ABI is the reason linking works at all.**
This chapter covers the System V AMD64 ABI in operational detail.

---

## 2.4.1 Why the ABI is sacred

When `a.o` calls a function defined in `b.o`, the two were compiled separately —
maybe by different compilers, in different years. They agree only because both
obey the same ABI. Violate it (mismatched calling convention, struct packing,
or C++ ABI) and you get silent corruption, not a clean error.

```
   a.cpp  ──compile──▶ a.o ┐
                           ├──link──▶ program     works ONLY because both
   b.cpp  ──compile──▶ b.o ┘                      sides obey the SAME ABI
```

The System V AMD64 ABI is the governing document on Linux/BSD/macOS for x86-64.
Windows x64 uses a *different* convention (covered briefly in §2.4.8).

---

## 2.4.2 Integer/pointer argument registers

The first six integer/pointer arguments go in registers, left to right:

```
   arg1   arg2   arg3   arg4   arg5   arg6   arg7+   →  on the STACK
   ────   ────   ────   ────   ────   ────   ─────────────────────────
   RDI    RSI    RDX    RCX    R8     R9     pushed right-to-left
```

Mnemonic: **"Diane's Silk Dress Cost $89"** → **D**I **S**I **D**X **C**X
**8 9** (R8, R9).

```
   void f(long a, long b, long c, long d, long e, long g, long h);
                 │     │     │     │     │     │     └ h → [rsp+0] (7th)
                 │     │     │     │     │     └ g → R9
                 │     │     │     │     └ e → R8
                 │     │     │     └ d → RCX
                 │     │     └ c → RDX
                 │     └ b → RSI
                 └ a → RDI
```

## 2.4.3 Floating-point arguments & return

Float/double arguments use **XMM0–XMM7** (separate counter from integers).
Return values:

```
   Integer/pointer return  →  RAX  (and RDX for 128-bit / two-word)
   Float/double return     →  XMM0 (and XMM1)
   __int128                 →  RAX:RDX
```

For **variadic** functions (`printf`), `AL` must hold the *number of vector
registers used* — that's why you see `mov eax, 0` before a `call printf` with no
float args.

> **C++ ▸** This is why calling a variadic function through a wrong prototype, or
> passing a `float` where `...` expects nothing special, can corrupt: the AL
> convention and the float-promotion-to-double rule must both hold.

---

## 2.4.4 Caller-saved vs callee-saved (volatility)

The single most important table for reading/writing assembly correctly:

```
 ┌─────────────────────────────────────────────────────────────────┐
 │ CALLEE-SAVED (preserved across calls — a function must restore  │
 │ them if it uses them):                                          │
 │     RBX  RBP  R12  R13  R14  R15   (and RSP, implicitly)        │
 ├─────────────────────────────────────────────────────────────────┤
 │ CALLER-SAVED (scratch — a caller must save them before a call   │
 │ if it needs them afterward):                                    │
 │     RAX RCX RDX RSI RDI R8 R9 R10 R11                           │
 │     all XMM/YMM/ZMM registers                                   │
 └─────────────────────────────────────────────────────────────────┘
```

```
   caller                          callee
   ──────                          ──────
   (RBX safe across call) ────────▶ if callee uses RBX it must
                                    push rbx ... pop rbx
   needs RCX after call?           callee may freely clobber RCX,
   must save it itself  ◀───────── so caller spills it first
```

This division is a *performance compromise*: callee-saved regs survive calls
(good for long-lived values) but cost save/restore; caller-saved are free to
clobber (good for temporaries).

> **War story ▸** Hand-written assembly that clobbers `RBX` without saving it
> "works" until the C++ caller happened to keep a loop counter there — then
> mysterious corruption. The ABI told you: RBX is callee-saved.

---

## 2.4.5 The stack frame, precisely

Putting §1.1.6 into ABI terms. A typical (non-omitted-frame-pointer) frame:

```
   Higher addresses
   ┌──────────────────────────┐
   │  ... caller's locals ... │
   ├──────────────────────────┤
   │  arg 8                   │  ┐ stack args, if any,
   │  arg 7                   │  ┘ pushed right-to-left by caller
   ├──────────────────────────┤
   │  return address          │  ← pushed by `call`        [rbp+8]
   ├──────────────────────────┤
   │  saved RBP               │  ← `push rbp; mov rbp,rsp`  [rbp+0] ◀─ RBP
   ├──────────────────────────┤
   │  local variables         │  [rbp-8], [rbp-16], ...
   │  saved callee regs       │
   │  outgoing arg area       │
   ├──────────────────────────┤  ◀─ RSP (16-byte aligned before next call)
   │  red zone (128 bytes)    │  scratch for leaf functions, below RSP
   └──────────────────────────┘
   Lower addresses
```

Standard prologue / epilogue:

```asm
 f:                          ; on entry: RSP % 16 == 8 (call pushed 8)
     push    rbp             ; save caller's frame pointer
     mov     rbp, rsp        ; establish our frame pointer
     sub     rsp, 32         ; allocate 32 bytes of locals (keeps alignment)
     ...                     ; body; locals at [rbp-8].. ; args at [rbp+16]..
     leave                   ; == mov rsp, rbp ; pop rbp
     ret
```

> **Frame-pointer omission ▸** With `-O2`, GCC/Clang often **omit RBP** as a
> frame pointer (using it as a general register and addressing locals off RSP).
> This saves an instruction and a register but means stack unwinders can't just
> "follow saved RBP" — they need **CFI** in `.eh_frame` (Part 6.4). Use
> `-fno-omit-frame-pointer` to keep RBP chains (helpful for profilers like
> `perf`).

---

## 2.4.6 Passing and returning structs (the gnarly part)

This is where ABIs get subtle and where C++ engineers get surprised. The System
V ABI **classifies** each eightbyte (8-byte chunk) of an aggregate as INTEGER,
SSE, or MEMORY, then assigns registers accordingly.

```
   struct Small { int a; int b; };        // 8 bytes, one INTEGER eightbyte
       → passed in a single register (RDI), returned in RAX

   struct Point { double x; double y; };  // 16 bytes, two SSE eightbytes
       → passed in XMM0, XMM1; returned in XMM0:XMM1

   struct Big { long a,b,c; };            // 24 bytes > 16  → MEMORY class
       → passed via a hidden pointer; caller allocates space, passes
         its address in RDI ("sret"); callee writes through it
```

```
   Return of a large struct (the "sret" / hidden first argument):

   T big_make();        becomes effectively   void big_make(T* hidden_ret);

   caller:   lea rdi, [rsp+slot]   ; address of caller's return slot
             call big_make         ; callee fills *rdi, returns rdi in rax
```

Rules of thumb:

- Aggregates ≤ 16 bytes are passed in up to two registers, classified per
  eightbyte (INTEGER→GPR, SSE→XMM).
- Aggregates > 16 bytes, or containing unaligned fields, are passed in **memory**.
- Non-trivial C++ types (non-trivial copy/move/dtor) are **always** passed in
  memory by *invisible reference* per the Itanium C++ ABI — regardless of size.

> **C++ ▸** This is exactly why `struct S { std::string s; };` is passed
> differently from `struct S { char buf[16]; };` even though both might be 16 or
> 32 bytes: the former is non-trivial, so it goes through memory with the caller
> constructing it. Understanding this explains a lot of "why is there a hidden
> pointer in my disassembly."

---

## 2.4.7 The full register-role map

```
 ┌────────┬────────────────────────────────────┬────────────┐
 │ Reg    │ Role                               │ Saved by   │
 ├────────┼────────────────────────────────────┼────────────┤
 │ RAX    │ return val (1st);AL=vec cnt vaargs │ caller     │
 │ RBX    │ general                            │ CALLEE     │
 │ RCX    │ arg4                               │ caller     │
 │ RDX    │ arg3; return val (2nd)             │ caller     │
 │ RSI    │ arg2                               │ caller     │
 │ RDI    │ arg1                               │ caller     │
 │ RBP    │ frame pointer (or general)         │ CALLEE     │
 │ RSP    │ stack pointer                      │ CALLEE     │
 │ R8     │ arg5                               │ caller     │
 │ R9     │ arg6                               │ caller     │
 │ R10    │ static chain ptr / scratch         │ caller     │
 │ R11    │ scratch                            │ caller     │
 │ R12-15 │ general                            │ CALLEE     │
 │ XMM0-1 │ FP args / FP return                │ caller     │
 │ XMM2-7 │ FP args                            │ caller     │
 │ XMM8-15│ scratch                            │ caller     │
 └────────┴────────────────────────────────────┴────────────┘
```

---

## 2.4.8 Other ABIs you'll encounter

### Windows x64

```
   Integer args:  RCX, RDX, R8, R9   (only 4!)   then stack
   FP args:       XMM0-3
   Callee-saved:  RBX RBP RDI RSI RSP R12-R15 + XMM6-15
   Shadow space:  caller reserves 32 bytes on stack for the 4 reg args
   No red zone.
```

Mixing System V and Windows conventions (e.g. naive FFI) is a classic crash. The
calling convention is part of the function's *type* in some compilers
(`__attribute__((sysv_abi))`, `__attribute__((ms_abi))`).

### AArch64 (AAPCS64)

```
   Integer/ptr args:  X0..X7        return: X0 (X1 for 128-bit)
   FP/SIMD args:      V0..V7        return: V0
   Callee-saved:      X19..X28, SP, and the low 64 bits of V8..V15
   Link register:     X30 (LR) holds return address (no pushed ret addr)
   Frame pointer:     X29 (FP)
   Indirect result:   X8 (the "sret" pointer register)
```

### 32-bit x86 cdecl (historical but still seen)

```
   ALL args on the stack, pushed right-to-left.
   Return in EAX (EDX:EAX for 64-bit). Caller cleans the stack (cdecl).
   stdcall: callee cleans the stack. fastcall: first 2 args in ECX,EDX.
```

---

## 2.4.9 Seeing it all in practice

**Try it ▸** Watch argument passing and struct return:

```bash
cat > abi.c <<'EOF'
struct Pt { double x, y; };
struct Big { long a,b,c; };
double  use(int a,int b,int c,int d,int e,int f,int g,double h){return a+b+g+h;}
struct Pt  mkpt(double x,double y){ struct Pt p={x,y}; return p; }
struct Big mkbig(long a){ struct Big b={a,a,a}; return b; }
EOF
gcc -O1 -S -masm=intel abi.c -o abi.s
# Read abi.s:
#  - `use`  : g comes from [rsp+...], h in xmm0, result in xmm0
#  - `mkpt` : returns in xmm0/xmm1 (two SSE eightbytes)
#  - `mkbig`: takes a hidden pointer in rdi, writes through it (MEMORY class)
```

**Try it ▸** Confirm callee-saved discipline:

```bash
printf 'long g();long f(long x){return g()+x;}\n' > cs.c
gcc -O2 -S -masm=intel cs.c -o cs.s
# `f` must keep `x` across the call to g(); watch it move x into a
# callee-saved register (rbx) and push/pop rbx around the call.
```

---

## Summary

- The ABI is the binary contract enabling separate compilation and linking;
  violating it causes silent corruption, not link errors.
- System V x86-64: integer args in `RDI,RSI,RDX,RCX,R8,R9`; FP in `XMM0–7`;
  return in `RAX`/`XMM0`; `AL` = vector count for varargs.
- Know the **caller-saved vs callee-saved** split cold (callee-saved: RBX, RBP,
  R12–R15).
- Aggregate passing is by eightbyte classification (≤16B in regs, >16B or
  non-trivial C++ via memory/hidden pointer = "sret").
- Frame-pointer omission at `-O2` is why stack unwinding needs CFI.
- Windows x64 and AArch64 differ in registers and details but follow the same
  principles.

This completes Part 2. → Part 3 dissects the tool that turns this assembly into
bytes: [3.1 — How an assembler works](../03-assembler/01-how-assemblers-work.md)
