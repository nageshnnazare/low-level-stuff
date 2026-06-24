# Assembly Language and Assemblers: A Deep Tutorial

This document is a detailed, practical introduction to assembly language and the tools that translate it into machine code. Examples target **x86-64** (AMD64) on **Linux** unless noted, using **NASM**-style Intel syntax for most listings because it maps cleanly to documentation and many tutorials. Where it matters, **GAS** (AT&T) is shown for comparison.

---

## 1. What Is Assembly Language?

### 1.1 History and Motivation

**Machine code** is the bit pattern the CPU decodes and executes directly. Early programmers wrote programs in raw numeric form (octal or hexadecimal), which was error-prone and hard to read.

**Assembly language** introduced *mnemonics* (human-readable tokens like `MOV`, `ADD`, `CALL`) and *symbolic labels* (names for addresses). A program called an **assembler** converts assembly into machine code. This made low-level programming feasible before high-level languages matured.

Historically:
- **1940s–1950s**: Programs toggled in or entered numerically; first assemblers appeared as simple mechanical substitutions.
- **1950s–1960s**: Macro assemblers, separate assembly *passes*, and linkers became standard.
- **Today**: Assembly is still used for OS kernels, bootloaders, performance-critical inner loops, embedded firmware, exploit research, reverse engineering, and teaching computer architecture.

**Motivation today**: Assembly gives *precise* control over registers, instruction selection, and calling conventions. It also makes visible what compilers usually hide: cache behavior, pipeline hazards at the microarchitectural level (when you profile), and ABI contracts.

### 1.2 Relationship Among Assembly, Machine Code, and High-Level Languages

| Layer | What it is | Executed by | Typical artifact |
|-------|------------|-------------|------------------|
| **High-level** (C, Rust, Go) | Abstract syntax, types, optimizations | — | `.c`, `.rs` |
| **Compiler IR / bytecode** | Internal representation (LLVM IR, JVM bytecode) | VM or compiler backend | `.ll`, `.class` |
| **Assembly** | Mnemonics + directives for one ISA | — | `.s`, `.asm` |
| **Machine code** | Encoded instructions + data | **CPU** | Bytes in `.o` / `.exe` |
| **Object / executable** | Machine code + metadata (relocations, symbols) | **Loader** | ELF, PE, Mach-O |

**Flow (compile → assemble → link → run):**

```
  +-------------------+     +-------------------+     +-------------------+
  |  Source (.c)      |     |  Assembly (.s)    |     |  Object (.o)      |
  |  int add(int a) { | --> |  add:             | --> |  (machine bytes   |
  |    return a+1;    |     |    lea eax,[rdi+1]|     |   + relocations)  |
  |  }                |     |    ret            |     +---------+---------+
  +-------------------+     +-------------------+               |
         |gcc -S|                                               | ld |
         v                                                      v
  +-------------------+                                    +-------------------+
  |  Optional:        |                                    |  Executable       |
  |  write asm by hand|----------------------------------->|  (ELF binary)     |
  +-------------------+                                    +---------+---------+
                                                                     |
                                                                     v
                                                          +-------------------+
                                                          |  OS loader maps   |
                                                          |  into memory;     |
                                                          |  CPU fetches      |
                                                          |  instructions     |
                                                          +-------------------+
```

**Key idea**: A **high-level** language expresses *intent* (`a + b`). **Assembly** expresses *particular* instructions on *particular* operands (which register, which memory addressing mode). **Machine code** is the encoding of those instructions. Many different high-level programs can compile to similar assembly; conversely, small assembly changes can map to very different machine encodings (e.g., shorter encodings for `eax` vs `rax` in 64-bit mode).

### 1.3 Language Hierarchy (ASCII)

```
                         +---------------------------+
                         |  Problem domain languages |
                         |  (SQL, HTML, DSLs, ...)   |
                         +-------------+-------------+
                                       |
                         +-------------v-------------+
                         |  General-purpose HLL      |
                         |  C, C++, Rust, Go, ...    |
                         +-------------+-------------+
                                       |
              +------------------------+------------------------+
              |                        |                        |
   +----------v----------+  +---------v---------+  +-----------v-----------+
   | Managed / VM        |  | AOT compiled      |  | Interpreted / JIT     |
   | (Java bytecode,     |  | to machine code   |  | (Python, JS engines)  |
   |  CIL, ...)          |  |                   |  |                       |
   +----------+----------+  +---------+---------+  +-----------+-----------+
              |                        |                        |
              +------------------------+------------------------+
                                       |
                         +-------------v-------------+
                         |  Assembly language        |
                         |  (ISA-specific)           |
                         +-------------+-------------+
                                       |
                         +-------------v-------------+
                         |  Machine code (ISA)       |
                         |  + relocations, debug     |
                         +-------------+-------------+
                                       |
                         +-------------v-------------+
                         |  Microarchitecture        |
                         |  (µops, execution units)  |
                         +---------------------------+
```

Assembly sits **one step above** raw machine code: it is *human-oriented* syntax for the same ISA.

---

## 2. Assembly Language Fundamentals (x86-64)

### 2.1 Registers: General Purpose, Special Purpose, Flags

#### General-purpose registers (64-bit names)

x86-64 extends the classic 32-bit `eax`… family to 64 bits (`rax`…). You can access **sub-registers**:

- **64-bit**: `rax`, `rbx`, `rcx`, `rdx`, `rsi`, `rdi`, `rbp`, `rsp`, `r8`–`r15`
- **32-bit**: `eax`, `ebx`, … (writes usually **zero-extend** to 64 bits in 64-bit mode)
- **16-bit**: `ax`, `bx`, …
- **8-bit**: `al`/`ah`, `bl`/`bh`, … (legacy split); `r8b`–`r15b` for extensions

**Convention (System V AMD64, Linux)**: Integer/pointer args often use `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` in order; return in `rax`. `rbx`, `rbp`, `r12`–`r15` are **callee-saved** (called functions must preserve them). `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`–`r11` are mostly **caller-saved** (except specific roles like `rcx` for some string ops—consult ABI for glue code).

#### Special-purpose registers

- **`rsp`**: Stack pointer. Must stay 16-byte aligned before `call` in SysV AMD64 (after `call`, return address makes misalignment unless adjusted).
- **`rbp`**: Frame pointer (optional in modern code with **omit-frame-pointer** optimization).
- **`rip`**: Instruction pointer (program counter). Not a general operand in classic Intel syntax except via **RIP-relative** addressing (`[rel label]` in NASM).
- **Segment registers** (`cs`, `ds`, `es`, `ss`, `fs`, `gs`): Mostly flat on Linux x86-64; `fs`/`gs` may hold TLS bases.

#### Flags register (`rFLAGS`)

Common flags (subset):

| Flag | Meaning (short) |
|------|------------------|
| **CF** | Carry (unsigned overflow/borrow) |
| **ZF** | Zero (result was zero) |
| **SF** | Sign (result negative in two's complement) |
| **OF** | Overflow (signed overflow) |
| **DF** | Direction (string ops increment/decrement index regs) |

**Compare** (`cmp`) and **test** (`test`) set flags without storing a “result” in a general register (they compute temporarily for flag update).

### 2.2 ASCII Diagram: x86-64 Register Layout (conceptual)

```
  64-bit register RAX (example)
   MSB                                                LSB
   +----+----+----+----+----+----+----+----+----+---+--+
   |                         64-bit RAX                |
   +-------------------------+-------------------------+
                             |        32-bit EAX       |
                             +------------+------------+
                                          | 16-bit AX  |
                                          +-----+------+
                                          | AH  | AL   |
                                          +-----+------+
                                           8    8   bits

  GPR map (names only; not to scale):

        +-------------------------------------------------------------+
        | RAX (return / accumulator)   RBX (often callee-saved)       |
        | RCX (arg4 / shift count)     RDX (arg3 / mul/div helper)    |
        | RSI (arg2 / string src)      RDI (arg1 / string dest)       |
        | RBP (frame pointer, opt.)    RSP (stack pointer)            |
        | R8–R15 (more GPRs; ABI defines saved vs volatile)           |
        +-------------------------------------------------------------+

        +-------------------------------+
        | RIP  (instruction pointer)    |
        | RFLAGS (status/control flags) |
        +-------------------------------+
```

### 2.3 Memory Addressing Modes

In **Intel syntax** (NASM), a memory operand often looks like:

```text
[ base + index*scale + displacement ]
```

**Examples (conceptual meanings):**

| Mode | Example (Intel) | Meaning |
|------|-----------------|--------|
| **Immediate** | `mov rax, 42` | Operand is constant in instruction |
| **Register** | `mov rax, rbx` | Operand in register |
| **Direct / absolute** | `mov rax, [qword 0x123456]` | Fixed address (rare in PIE) |
| **Indirect** | `mov rax, [rbx]` | Address in register |
| **Indexed** | `mov rax, [rbx+rcx*8]` | Base + scaled index |
| **RIP-relative** | `mov eax, [rel my_var]` | PC-relative (position-independent) |

**ASCII: addressing pieces**

```
  instruction references memory at address:

       displacement  (constant offset, optional)
              |
              v
         +----+----------------------------------+
         |    |  base reg  +  (index reg * scale)|
         +----+----------------------------------+
              \_____________address______________/
```

**Scale** is typically 1, 2, 4, or 8 (matches element sizes).

### 2.4 Data Types and Sizes

**Integral sizes** (common NASM sizing):

| NASM keyword | Size | Bytes |
|--------------|------|-------|
| `byte` | 8-bit | 1 |
| `word` | 16-bit | 2 |
| `dword` | 32-bit | 4 |
| `qword` | 64-bit | 8 |
| `oword` | 128-bit (SSE) | 16 |

Use correct size in memory operands: `mov byte [rdi], 0` vs `mov qword [rdi], 0`.

### 2.5 Sections: `.text`, `.data`, `.bss`, `.rodata`

Typical ELF sections for hand-written assembly:

```
  +----------+----------------------------------------+------------------+
  | Section  | Typical contents                       | Writable?        |
  +----------+----------------------------------------+------------------+
  | .text    | Machine instructions (code)            | No (exec + read) |
  | .rodata  | Constants, string literals             | No (read only)   |
  | .data    | Initialized mutable globals            | Yes              |
  | .bss     | Zero-initialized mutable (no disk img) | Yes              |
  +----------+----------------------------------------+------------------+
```

**ASCII: process image (simplified)**

```
  Low addresses
     |
     |   + .text (code)
     |   + .rodata (strings, LUTs)
     |   + .data (initialized globals)
     |   + .bss  (zeroed globals)
     |   + heap ( grows --> )
     |
     |   ...
     |
     |   + stack ( <-- grows )
     v
  High addresses (traditional diagram; actual ASLR varies)
```

**NASM segment directives (ELF64):**

```nasm
section .text
section .rodata
section .data
section .bss
```

---

## 3. Instruction Categories

> **Note**: Instruction availability and operand rules vary by mode (16/32/64) and prefixes. The examples below assume **64-bit long mode** on Linux.

### 3.1 Data Movement

| Instruction | Role |
|-------------|------|
| **mov** | Copy bits between register/memory/immediate (not all combinations legal) |
| **lea** | **Load effective address** — computes address without memory access |
| **push** / **pop** | Stack push/pop (_RSP adjusts by 8 in 64-bit) |
| **xchg** | Exchange two operands |

**Example (NASM, Linux x86-64):**

```nasm
section .text
global _start

_start:
    mov     rax, 0              ; immediate -> rax (actually optimize: xor eax,eax for zero)
    xor     eax, eax            ; rax = 0 (idiomatic, shorter encoding often)

    mov     rbx, 0x1000
    lea     rcx, [rbx+8]        ; rcx = rbx + 8 (no memory read)

    push    rbx
    pop     rdx                 ; rdx now equals old rbx
    xchg    rbx, rdx            ; swap rbx and rdx

    mov     rax, 60             ; sys_exit
    xor     edi, edi            ; status 0
    syscall
```

**`lea` for “math”**: Compilers often use `lea` for add/multiply tricks because it can combine base+index*scale+offset in one instruction without touching flags the way `add` might.

### 3.2 Arithmetic

| Instruction | Role |
|-------------|------|
| **add** / **sub** | Add/subtract |
| **inc** / **dec** | Increment/decrement (partial flags update—be careful in tight flag-dependent sequences) |
| **neg** | Negate (two's complement) |
| **imul** / **mul** | Signed/unsigned multiply (forms vary: one-operand uses RAX/RDX) |
| **idiv** / **div** | Signed/unsigned divide (uses RDX:RAX) |

**Example (64-bit signed multiply carefulness):**

```nasm
section .text
global mult_demo
mult_demo:
    ; int64_t mult_demo(int64_t a, int64_t b)  — SysV: RDI=a, RSI=b

    mov     rax, rdi
    imul    rax, rsi            ; rax = a * b (two-operand form)
    ret
```

**Example (preparing for `idiv`):**

```nasm
; Divide 128-bit in rdx:rax by rcx; quotient in rax, remainder in rdx
; (Assumes values fit result — overflows trap #DE)

    cqo                         ; sign-extend rax into rdx
    idiv    rcx
```

### 3.3 Logic / Bitwise

| Instruction | Role |
|-------------|------|
| **and** / **or** / **xor** | Bitwise AND/OR/XOR |
| **not** | Bitwise NOT |
| **shl** / **shr** | Logical shifts |
| **sal** / **sar** | Arithmetic shifts |
| **rol** / **ror** | Rotates |

**Clear register idiom**: `xor eax, eax` (faster/smaller than `mov eax, 0` in many cases).

**Example:**

```nasm
section .text
global bit_twiddle
bit_twiddle:
    mov     rax, rdi
    xor     rax, rsi          ; rax = a ^ b
    not     rax
    and     rax, 0xFF         ; keep low byte (watch operand sizes)
    ret
```

### 3.4 Control Flow

| Instruction | Role |
|-------------|------|
| **jmp** | Unconditional jump |
| **j*** | Conditional jumps (test flags: `je`, `jne`, `jg`, `jl`, `ja`, `jb`, …) |
| **cmp** / **test** | Set flags for branches |
| **call** / **ret** | Function call/return |
| **loop*** | Decrement RCX and branch (legacy; rarely used in modern 64-bit code) |

**Example:**

```nasm
section .text
global max_abi
max_abi:
    cmp     rdi, rsi
    jge     .rdi_wins
    mov     rax, rsi
    ret
.rdi_wins:
    mov     rax, rdi
    ret
```

### 3.5 String / Memory Block Primitives

**REP prefixes** repeat string instructions `RCX` times (subject to **DF** direction flag). Common instructions:

| Mnemonic | Effect (conceptual) |
|----------|---------------------|
| **movsb/w/d/q** | Move string element from `[RSI]` to `[RDI]`, advance pointers |
| **cmpsb/w/d/q** | Compare elements, advance pointers |
| **scasb/w/d/q** | Compare `[RDI]` with AL/AX/EAX/RAX, advance RDI |
| **stosb/w/d/q** | Store AL/… into `[RDI]`, advance RDI |
| **lodsb/w/d/q** | Load from `[RSI]` into AL/…, advance RSI |

**Example (NASM): POSIX `memset`-shaped routine with `rep stosb`**

SysV AMD64 Linux: `void *memset(void *rdi_s, int esi_c, size_t rdx_n);` — fill byte is **the low 8 bits of `ESI`**.

```nasm
section .text
global asm_memset
asm_memset:
    mov     r8, rdi             ; save original s (rep stosb advances rdi)
    mov     rcx, rdx            ; count
    movzx   eax, sil            ; fill byte from sil (lower 8 bits of 2nd arg)
    cld
    rep stosb
    mov     rax, r8
    ret
```

**Example: copy a fixed-size block with `rep movsq`**

The hardware implicitly reads **`RSI`** (source), writes **`RDI`** (destination), and advances both by 8 when **`RCX`** counts down. This snippet is **not** a full SysV function by itself—you must load **`RDI`**/ **`RSI`** before calling it (shown as a labeled block you might `call` after setting registers yourself):

```nasm
section .text
global copy_16qwords
copy_16qwords:
    ; Preconditions: RDI = dest, RSI = src, direction flag clear
    mov     rcx, 16
    cld
    rep movsq
    ret
```

*Production* `memset`/`memcpy` in libc is heavily optimized (SIMD, alignment prologues, microcoded **`rep movsb`** on CPUs that advertise **ERMSB** / “fast short **`rep mov`**” style features, etc.). Hand-written `rep` loops remain excellent for recognizing compiler output and for **tiny** fixed regions.

**`cmpsb` + `repe` / `repne`:** repeat while **`ZF`** matches the prefix condition—`repe cmpsb` compares until **mismatch** or **`RCX=0`**. This mirrors **`memcmp`-style** loops in microcode form:

```nasm
; int cmp_prefix(const char *a, const char *b, size_t n);
; SysV: RDI=a, RSI=b, RDX=n — clobbers RCX, RDI, RSI per `repe cmpsb`
section .text
global cmp_prefix
cmp_prefix:
    mov     rcx, rdx
    cld
    repe cmpsb
    jz      .equal
    mov     eax, 1
    ret
.equal:
    xor     eax, eax
    ret
```

**`loope` / `loopne` / `loop`**: the `loop` instruction uses **`RCX`** in 64-bit mode (not ECX). It decrements RCX and jumps if RCX ≠ 0; `loope` / `loopne` also consult ZF. Compilers almost never emit `loop` on x86-64 because it is slow on contemporary cores; a **dec/jnz** pair is normally faster.

### 3.6 System Calls

**Linux x86-64** uses the **`syscall`** instruction with:

- **Syscall number** in `RAX`
- Arguments in **`RDI`, `RSI`, `RDX`, `R10`, `R8`, `R9`** (note `R10` instead of `RCX`)
- Return value in `RAX`; on error, negated errno (convention: `-errno`)

**Legacy Linux ia32 (32-bit)** used **`int 0x80`** with this register mapping (classic):

| Argument | Register |
|---------|-----------|
| syscall # | `EAX` |
| 1st | `EBX` |
| 2nd | `ECX` |
| 3rd | `EDX` |
| 4th | `ESI` |
| 5th | `EDI` |
| 6th | `EBP` |

**Do not** use `int 0x80` from **64-bit** long-mode code for Linux system calls: the kernel may treat it as a **32-bit compatibility** syscall path or behave inconsistently relative to native `syscall`. Treat **`syscall` + `/usr/include/asm/unistd_64.h` numbers** as the supported interface for new x86-64 programs.

**Historical / classroom ia32 sketch (assemble as 32-bit ELF):**

```nasm
; hello_int0x80_32.asm — Linux i386 only
;   nasm -f elf32 hello_int0x80_32.asm -o h32.o && ld -m elf_i386 h32.o -o h32

section .text
global _start

_start:
    mov     eax, 4              ; sys_write (ia32)
    mov     ebx, 1
    mov     ecx, msg
    mov     edx, msg_len
    int     0x80

    mov     eax, 1              ; sys_exit
    xor     ebx, ebx
    int     0x80

section .rodata
msg:        db "Hello, int 0x80", 10
msg_len:    equ $ - msg
```

**Example: write(1, msg, len) + exit(0)** (x86-64 `syscall`)

```nasm
; hello_syscall.asm — assemble and link:
;   nasm -f elf64 hello_syscall.asm -o hello_syscall.o
;   ld hello_syscall.o -o hello_syscall

section .rodata
msg: db "Hello, syscall", 10
len: equ $ - msg

section .text
global _start

_start:
    mov     rax, 1              ; sys_write
    mov     rdi, 1              ; fd stdout
    mov     rsi, msg
    mov     rdx, len
    syscall

    mov     rax, 60             ; sys_exit
    xor     edi, edi
    syscall
```

---

## 4. The Assembler

### 4.1 What an Assembler Does

An assembler performs roughly:

1. **Tokenize** source lines (mnemonics, operands, labels, directives).
2. **Build symbol table** (label → address or value).
3. **Encode instructions** to bytes (may depend on address sizes, displacements).
4. **Emit sections** with relocations for the linker (unless absolute output).

### 4.2 Two-Pass Assembly (ASCII)

```
  PASS 1                         PASS 2
  ------                         ------

  Read source                    Read source again
       |                              |
       v                              v
  Assign tentative                Encode instructions using
  addresses; collect              final addresses & symbol values
  label definitions                    |
       |                               v
       v                         Emit object bytes +
  Resolve what you can            relocation entries
  (ORG/align effects)                  |
       |                               v
       v                         Linker merges objects,
  Symbol table (labels)          resolves externals,
                                 produces executable
```

**Why two passes?** Forward references: a branch to `label` that appears later needs the label’s final address. Pass 1 records label offsets; pass 2 fills in displacements.

**One-pass** assemblers exist with **backpatching** or restrictions.

### 4.3 Major Assemblers

| Assembler | Typical syntax | Common platforms | Notes |
|-----------|---------------|------------------|-------|
| **NASM** | Intel | Linux, Windows, macOS | Popular teaching & cross-platform |
| **GAS** | AT&T default; **`.intel_syntax`** available | Unix-like (GCC toolchain) | GNU assembler |
| **MASM** | Intel (Microsoft variant) | Windows | MSVC ecosystem |
| **FASM** | Intel-like | Cross-platform | Self-hosted, macro-heavy |

### 4.4 GAS AT&T vs Intel Side-by-Side

**Rule of thumb (AT&T)**:

- Source **then** dest: `mov %rsi, %rdi` moves from RSI into RDI.
- Registers prefixed `%`, immediates `$`.
- Memory: `disp(base, index, scale)`.

**Intel (NASM)**:

- Dest **then** source for `mov`: `mov rdi, rsi`.

```
  OPERAND ORDER (MOV example)

  Intel:     MOV   dest, src     ; rdi <- rsi
               \_____/   \__/
                 |        |
  AT&T:      mov   %src, %dest   ; mov %rsi, %rdi
```

**Example pair (same effect):**

```text
Intel (NASM):     add rax, rbx
AT&T (GAS):       add %rbx, %rax
```

**GAS Intel syntax snippet:**

```gas
.intel_syntax noprefix
mov rax, rbx
.att_syntax prefix
```

### 4.5 Directives and Pseudo-Instructions

**Directives** are assembler commands (not CPU instructions):

| NASM examples | Purpose |
|---------------|---------|
| `section .text` | Switch output section |
| `global _start` | Export symbol to linker |
| `extern printf` | Undefined external symbol |
| `db`, `dw`, `dd`, `dq` | Emit raw bytes/words/dwords/qwords |
| `equ` | Constant definition |
| `times n nop` | Repeat emission |
| `align 16` | Alignment |

**Pseudo-instructions**: Convenient forms expanded to real instructions (assembler-specific), e.g., GAS `call *%rax` forms, or NASM’s smart choices for shorter encodings when you use `default rel`.

### 4.6 Macro System (NASM example)

```nasm
%macro save_regs 0
    push    rbx
    push    r12
%endmacro

%macro restore_regs 0
    pop     r12
    pop     rbx
%endmacro
```

FASM and MASM have richer macro metaprogramming; GAS uses `.macro`.

---

## 5. Calling Conventions

### 5.1 System V AMD64 ABI (Linux, macOS, many Unixes)

**Integer/pointer arguments (userland):**

1. `RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9`
2. Additional arguments on the **stack** (right-to-left in C terms; ABI specifies alignment/red zone—read carefully for variadics).

**Return**: `RAX` (and `RDX` for 128-bit aggregates in some cases).

**Callee-saved**: `RBX`, `RBP`, `R12`–`R15` (must preserve across call if used).

**Caller-saved / scratch**: `RAX`, `RCX`, `RDX`, `RSI`, `RDI`, `R8`–`R11`.

**Stack alignment**: Before a `call`, `RSP % 16 == 0` (Linux AMD64 ABI).

**Red zone**: 128 bytes below `RSP` usable without moving stack (leaf functions—care with signal/async assumptions).

### 5.2 Windows x64 Calling Convention (Microsoft)

- First four **integer/pointer** arguments go in **`RCX`, `RDX`, `R8`, `R9`** (regardless of type size, subject to classification rules for aggregates).
- Fifth and later arguments go on the **stack** in **right-to-left** order; each parameter consumes **at least 8 bytes** of stack space.
- The caller provides a **32-byte “shadow store”** (home space) above the return address for the callee to spill the register arguments if needed; the caller **allocates** this space before each `call`.
- **`XMM0`–`XMM3`** pass floating-point arguments in parallel with the integer registers for the first four args (when applicable).
- The first 32 bytes of stack args must remain **16-byte aligned** relative to the ABI rules for varargs/vector-call surfaces—consult Microsoft’s docs for edge cases.

**ASCII: caller stack right before `call target(a,b,c,d,e)` (conceptual)**

```
      +--> higher addresses
      |
      |   [ space for 5th+ args, if any ]
      |   [ 32-byte shadow space for callee spill of RCX..R9 / XMM0..XMM3 ]
      |   [ return address will go here when CALL executes ]
      v
```

Different name mangling, structured exceptions (`SEH`), and metadata (`pdata`, `xdata`) separate this ABI from ELF/Linux SysV.

### 5.3 Stack Frame Layout (typical; ASCII)

```
 Higher addresses
      |
      |    prior frame ...
      +----------------------------+
      |  return address            | <-- [RSP] immediately after CALL
      +----------------------------+
      |  saved RBP (if using frame)| <-- typical prologue pushes rbp
      +----------------------------+     mov rbp, rsp
      |  local var 1               |
      +----------------------------+
      |  local var 0               |
      +----------------------------+
      v  RSP moves down for locals
 Lower addresses

  Typical prologue / epilogue (conceptual):

    push rbp
    mov  rbp, rsp
    sub  rsp, N        ; allocate locals

    ... body ...

    mov  rsp, rbp
    pop  rbp
    ret
```

### 5.4 Interfacing With C

**Declare in C:**

```c
int addnums(int a, int b);
```

**Implement in NASM (Linux ELF64):**

```nasm
section .text
global addnums
addnums:
    lea     eax, [rdi + rsi]    ; return 32-bit int in eax (zero-extends to rax)
    ret
```

**Build:**

```bash
nasm -f elf64 addnums.asm -o addnums.o
gcc -O2 main.c addnums.o -o prog
```

**C calling assembly with proper prototype** avoids ABI mismatches.

---

## 6. Practical Examples

### 6.1 Hello World (libc `printf` vs raw `syscall`)

**Using libc and `printf`:**

```c
/* main.c */
#include <stdio.h>
int main(void) {
    printf("Hello from C\n");
    return 0;
}
```

```nasm
; hello_printf.asm — link with gcc so libc + dynamic linker initialize the process
default rel

section .rodata
fmt: db "Hello from asm: %s", 10, 0
msg: db "via printf", 0

section .text
extern printf
global main

main:
    sub     rsp, 8              ; ensure RSP % 16 == 0 at the CALL boundary (§5.1)
    lea     rdi, [fmt]
    lea     rsi, [msg]
    xor     eax, eax            ; RAX = 0 XMM args used for this variadic call
    call    printf wrt ..plt

    xor     eax, eax
    add     rsp, 8
    ret
```

```bash
nasm -f elf64 hello_printf.asm -o hello_printf.o
gcc -pie hello_printf.o -o hello_printf
```

`default rel` makes bare symbols like `[fmt]` RIP-relative in NASM, which matches **PIE** executables. For a static non-PIE link you can often use absolute addresses instead, but PIE is the common distro default.

**Minimal `_start` + write syscall** (already in §3.6) avoids libc entirely.

### 6.2 Fibonacci Sequence

Iterative: \(F(0)=0\), \(F(1)=1\), \(F(n)=F(n-1)+F(n-2)\). For \(n \ge 2\), loop **\(n-1\)** times updating `(a,b) <- (b, a+b)`; result is **`b`**.

```nasm
; fib_iter.asm — uint64_t fib(uint64_t n)  (SysV: n in RDI)
section .text
global fib
fib:
    cmp     rdi, 1
    jbe     .base               ; n<=1 -> return n

    xor     r8, r8              ; a = 0  (64-bit; Fibonacci overflows quickly anyway)
    mov     r9, 1               ; b = 1
    lea     rcx, [rdi - 1]      ; iterations = n-1

.loop:
    lea     rax, [r8 + r9]      ; next = a + b (use rax temp, not rdi)
    mov     r8, r9              ; a = old b
    mov     r9, rax             ; b = next
    dec     rcx
    jnz     .loop

    mov     rax, r9
    ret

.base:
    mov     rax, rdi
    ret
```

**Avoid** clobbering **`RDI`** mid-function if it still holds the original **`n`** unless you have copied it elsewhere first.

### 6.3 String Manipulation (length with scalar loop)

```nasm
; strlen_asm.asm — size_t strlen_asm(const char* rdi)
section .text
global strlen_asm
strlen_asm:
    xor     rax, rax
.loop:
    cmp     byte [rdi + rax], 0
    je      .done
    inc     rax
    jmp     .loop
.done:
    ret
```

**Using `repne scasb` (classic idiom):** start with **`RCX = -1`**, **`AL = 0`**, **`CLD`**. Each `REPNE` iteration pre-decrements `RCX` and runs `SCASB` until `ZF=1` (NUL seen). If the string length is **L**, there are **L+1** comparisons, so **`RCX_final = -1 - (L+1) = -L-2`**. Hence **`L = -2 - RCX_final`** (equivalently the **`NOT RCX; DEC RCX`** tick you often see in older 32-bit write-ups).

```nasm
section .text
global strlen_scas
strlen_scas:
    xor     eax, eax
    mov     rcx, -1
    cld
    repne scasb
    mov     rax, -2
    sub     rax, rcx
    ret
```

On entry, **`RDI`** must point at the string (SysV first argument). Prefer **`strlen`** from libc in production.

### 6.4 Calling C Library Functions From Assembly

**`strdup` shape: `strlen` + `malloc` + `strcpy` (Linux, SysV AMD64, PIC):**

```nasm
default rel

section .text
extern strlen
extern malloc
extern strcpy
global asm_strdup

; char *asm_strdup(const char *s);  /* RDI = s */
asm_strdup:
    push    r12
    push    rbx
    mov     r12, rdi

    call    strlen wrt ..plt        ; RAX = len (not counting NUL)

    lea     rdi, [rax + 1]          ; strlen + 1 bytes
    call    malloc wrt ..plt
    test    rax, rax
    je      .fail

    mov     rbx, rax                ; dest buffer
    mov     rdi, rbx
    mov     rsi, r12
    call    strcpy wrt ..plt        ; returns dest in RAX per SysV ABI

    pop     rbx
    pop     r12
    ret

.fail:
    xor     eax, eax
    pop     rbx
    pop     r12
    ret
```

```bash
nasm -f elf64 asm_strdup.asm -o asm_strdup.o
gcc -pie main.c asm_strdup.o -o main
```

Link with **`gcc`** (not plain `ld`) when you rely on **`libc`** so initialization, IFUNC resolvers, and thread-local setup stay correct. Pure `_start` programs can use **`syscall`** instead (§3.6) if they avoid libc entirely.

---

## 7. Assembly in Modern Context

### 7.1 Inline Assembly in C/C++

**GCC/Clang extended asm:**

```c
#include <stdint.h>

uint32_t rotl32(uint32_t x, unsigned n) {
    __asm__(
        "roll %%cl, %0"
        : "+r"(x)
        : "c"(n)
        : /* clobber */
    );
    return x;
}
```

**`asm volatile`**: tells the compiler not to delete asm as “dead code” even if outputs appear unused—important for MMIO or timing-sensitive sequences.

** MSVC** uses `__asm` (historically 32-bit; 64-bit MSVC has no inline asm—use intrinsics instead).

### 7.2 When and Why to Use Assembly Today

**Reasonable uses:**

- Hand-tuned hot paths *after* profiling proves compiler misses (rare)
- OS trap/interrupt entry points, context switch glue
- Cryptographic constant-time considerations (still fraught—compilers may optimize “around” you)
- Bootstrapping environments without a compiler
- Security research (shellcode, ROP gadget cataloging—ethical/legal boundaries apply)

**Prefer first**: algorithmic improvements, better data layout, compiler intrinsics (`immintrin.h`), vectorization, threading.

### 7.3 SIMD Overview (SSE / AVX)

**SIMD**: Single Instruction, Multiple Data—one opcode operates on wide registers holding packed lanes.

- **SSE (XMM, 128-bit)**: `xmm0`–`xmm15` in 64-bit; packed floats/doubles/integers
- **AVX (YMM, 256-bit)** widens ops; **AVX-512 (ZMM)** further widens on supporting CPUs

**Representative families:**

| Family | Example intent |
|--------|----------------|
| **Move / broadcast** | `movdqa`, `vbroadcastss` |
| **Arithmetic** | `addps`, `vaddpd` |
| **Compare / mask** | `cmpps`, AVX-512 mask registers |
| **Shuffle / permute** | `shufps`, `vpermilps` |

**C intrinsics** usually beat hand assembly for portability:

```c
#include <immintrin.h>

void add_four_floats(const float* a, const float* b, float* out) {
    __m128 va = _mm_loadu_ps(a);
    __m128 vb = _mm_loadu_ps(b);
    __m128 vc = _mm_add_ps(va, vb);
    _mm_storeu_ps(out, vc);
}
```

---

## Appendix A: Quick x86-64 Linux Syscall Numbers (Partial)

| RAX | Name |
|-----|------|
| 0 | read |
| 1 | write |
| 60 | exit |

Always verify against `unistd_64.h` on your distro/kernel docs—numbers can evolve with architectures.

---

## Appendix B: Minimal Build Commands Cheatsheet

```bash
# NASM object + static-ish link with gcc (brings CRT)
nasm -f elf64 file.asm -o file.o
gcc file.o -o file

# Pure syscall binary (no libc)
nasm -f elf64 raw.asm -o raw.o
ld raw.o -o raw

# GAS
gcc -c file.S -o file.o
```

---

## Further Reading

- Intel® 64 and IA-32 Architectures Software Developer’s Manuals (volumes 1–3)
- System V Application Binary Interface (AMD64 Architecture Processor Supplement)
- AMD64 Architecture Programmer’s Manual
- Agner Fog’s optimization manuals (microarchitecture)

---

*End of tutorial.*
