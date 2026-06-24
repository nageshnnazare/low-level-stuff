# 1.1 — CPU, Registers & the Memory Model

Before we can talk about assemblers and linkers, we need a precise mental model
of the machine they target. This chapter builds that model from the silicon up,
but only as deep as the toolchain hacker needs.

---

## 1.1.1 The von Neumann machine you actually program

Every mainstream CPU you'll meet is, at the programmer level, a **stored-program
von Neumann machine**: code and data live in the same address space, and the CPU
executes a fetch-decode-execute loop.

```
            ┌──────────────────────────────────────────────┐
            │                    CPU CORE                  │
            │                                              │
            │   ┌─────────┐   ┌──────────┐   ┌─────────┐   │
            │   │  Fetch  │──▶│  Decode  │──▶│ Execute │   │
            │   └────▲────┘   └──────────┘   └────┬────┘   │
            │        │                            │        │
            │   ┌────┴─────────────────┐    ┌─────▼─────┐  │
            │   │ RIP (program counter)│    │ Registers │  │
            │   └──────────────────────┘    │ ALU / FPU │  │
            │                               └─────┬─────┘  │
            └─────────────────────────────────────┼────────┘
                       ▲   load / store           │
                       │                          ▼
            ┌──────────┴────────────────────────────────────┐
            │   MEMORY (one flat array of bytes)            │
            │   addr 0 ............................. 2^64-1 │
            │   [ code ][ rodata ][ data ][ heap → ][ ← stk]│
            └───────────────────────────────────────────────┘
```

The CPU repeatedly:

1. **Fetches** the bytes at the address in the instruction pointer (`RIP` on
   x86-64, `PC` on ARM).
2. **Decodes** those bytes into an operation + operands.
3. **Executes** it (arithmetic, a load/store, a branch...).
4. Advances `RIP` to the next instruction (unless the instruction was a branch).

Everything the assembler emits and the linker arranges is ultimately *bytes in
that flat memory array*, destined to be fetched by this loop.

> **Reality check ▸** Modern cores are wildly out-of-order, superscalar,
> speculative, and pipelined. But the *architectural* (programmer-visible)
> model is still this simple sequential machine. The micro-architecture is an
> implementation detail that preserves the illusion of sequential execution.
> For the toolchain, the architectural model is what matters.

---

## 1.1.2 Registers: the CPU's working set

Registers are the tiny, ultra-fast storage locations *inside* the CPU. On
x86-64 there are 16 general-purpose 64-bit integer registers, plus the
instruction pointer, flags, vector registers, and more.

### General-purpose registers (GPRs) on x86-64

```
 63                              31          15      7     0
┌───────────────────────────────┬───────────┬────────┬──────┐
│                               RAX                         │  64-bit
│                               ┌───────────────────────────┤
│                               │           EAX             │  32-bit
│                               │           ┌───────────────┤
│                               │           │      AX       │  16-bit
│                               │           │   ┌────┬──────┤
│                               │           │   │ AH │  AL  │  8-bit
└───────────────────────────────┴───────────┴───┴────┴──────┘
```

The same physical register is addressable at 4 widths. This sub-register
aliasing is a frequent source of bugs and a key thing assemblers and
disassemblers must encode precisely.

| 64-bit | 32-bit | 16-bit | 8-bit | Conventional role (System V)        |
|--------|--------|--------|-------|-------------------------------------|
| RAX    | EAX    | AX     | AL    | return value / accumulator          |
| RBX    | EBX    | BX     | BL    | callee-saved                        |
| RCX    | ECX    | CX     | CL    | 4th integer arg                     |
| RDX    | EDX    | DX     | DL    | 3rd integer arg / return high       |
| RSI    | ESI    | SI     | SIL   | 2nd integer arg                     |
| RDI    | EDI    | DI     | DIL   | 1st integer arg                     |
| RBP    | EBP    | BP     | BPL   | frame pointer (callee-saved)        |
| RSP    | ESP    | SP     | SPL   | **stack pointer**                   |
| R8–R15 | R8D…   | R8W…   | R8B…  | args 5–6 (R8,R9) + scratch/saved    |

> **Quirk ▸** Writing to a 32-bit register (e.g. `mov eax, 1`) **zero-extends**
> into the full 64-bit register (RAX becomes `0x0000_0000_0000_0001`). But
> writing to a 16- or 8-bit sub-register (`mov al, 1`) leaves the upper bits
> **unchanged**. This asymmetry exists for performance (avoiding partial-register
> stalls) and is something disassemblers and the ABI both rely on.

### Special registers

| Register | Purpose                                                          |
|----------|------------------------------------------------------------------|
| `RIP`    | Instruction pointer. Not directly writable except via branches.  |
| `RFLAGS` | Status flags: CF, ZF, SF, OF, PF, AF, DF, etc.                   |
| `RSP`    | Stack pointer (grows *downward* toward lower addresses).         |
| `CS,DS,SS,ES,FS,GS` | Segment selectors. Mostly vestigial in 64-bit, **except** `FS`/`GS` which are repurposed as bases for thread-local storage (see Part 7.1). |
| `XMM0–15`/`YMM`/`ZMM` | SIMD / floating-point vector registers (SSE/AVX/AVX-512). XMM0–7 also carry float/double args & returns. |

The `FS`/`GS` detail matters enormously later: **thread-local storage** is
implemented by pointing `FS` (Linux) or `GS` (Windows/macOS) at a per-thread
control block. We'll see the assembler emit `%fs:` prefixes and special
relocations for it.

---

## 1.1.3 The flags register

Arithmetic and logic instructions set bits in `RFLAGS`. Branches read them.

```
 RFLAGS (low 16 bits shown)
 15        11   10   9   8   7   6      4      2    0
┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│..│OF│DF│IF│TF│SF│ZF│..│AF│..│PF│..│CF│  ...   │
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
   CF = Carry      ZF = Zero      SF = Sign
   OF = Overflow   PF = Parity    DF = Direction (string ops)
```

The pattern `cmp a, b` followed by `je`/`jl`/`jg` works by subtracting and
setting flags, then branching on those flags. The compiler chooses the right
conditional jump based on signedness — a detail that surfaces in disassembly.

---

## 1.1.4 The flat virtual address space

A 64-bit process *sees* a single, flat array of bytes from address `0` to
`2^64 - 1`. In practice current x86-64 hardware implements 48-bit (or 57-bit
with 5-level paging) canonical addresses, so the usable space is split:

```
0xFFFFFFFFFFFFFFFF ┌──────────────────────────────┐
                   │      kernel space            │  (not user-accessible)
0xFFFF800000000000 ├──────────────────────────────┤
                   │   non-canonical "hole"       │  (faults if accessed)
0x00007FFFFFFFFFFF ├──────────────────────────────┤
                   │      user space (128 TiB)    │
0x0000000000000000 └──────────────────────────────┘
```

The MMU translates these **virtual addresses** to **physical addresses** via
page tables, transparently to your program. The linker and loader care deeply
about virtual addresses: the linker assigns them, the loader maps file contents
to them.

> **Key insight ▸** When the linker says a symbol `main` is at address
> `0x1149`, that is a *virtual* address. The OS guarantees that when your code
> runs, that virtual address contains your function — even though the physical
> RAM could be anywhere, and (with ASLR) the virtual base may be randomized.

---

## 1.1.5 The process memory layout

When the loader sets up a process, it carves virtual memory into regions
("segments" in the loader sense, "mappings" / VMAs in the kernel sense):

```
 high addr  0x7fff... ┌───────────────────────────┐
                      │  [stack]                  │  grows ↓  (locals, return
                      │      │                    │            addrs, saved regs)
                      │      ▼                    │
                      ├───────────────────────────┤
                      │      ▲                    │
                      │   (shared libs, mmap)     │  ← libc.so, ld.so map here
                      ├───────────────────────────┤
                      │      ▲                    │
                      │      │                    │
                      │  [heap]                   │  grows ↑  (malloc/new via
                      ├───────────────────────────┤            brk/mmap)
                      │  .bss   (zero-init data)  │
                      │  .data  (init data)       │  ← writable
                      ├───────────────────────────┤
                      │  .rodata (constants)      │  ← read-only
                      │  .text  (machine code)    │  ← read + execute
 low addr   0x400000  └───────────────────────────┘
                      │   (NULL guard page)       │  ← deref of nullptr faults
            0x000000  └───────────────────────────┘
```

Memorize this. Almost everything in Parts 4–7 is about *how the linker decides
what goes in which region* and *how the loader builds it*.

| Region    | Permissions | Backed by                  | Holds                          |
|-----------|-------------|----------------------------|--------------------------------|
| `.text`   | r-x         | file (demand-paged)        | machine code                   |
| `.rodata` | r--         | file                       | string literals, const tables, vtables-ish data |
| `.data`   | rw-         | file                       | initialized globals/statics    |
| `.bss`    | rw-         | zero-fill (no file bytes)  | zero-initialized globals       |
| heap      | rw-         | anonymous                  | `malloc`/`new`                 |
| stack     | rw-         | anonymous, grows down      | call frames                    |
| mmap libs | varies      | file (the `.so`)           | shared library code/data       |

> **C++ ▸** A `const char* s = "hello";` puts `"hello\0"` in `.rodata`. A
> `static int counter = 0;` goes in `.bss` (zero, no file bytes). A
> `static int x = 42;` goes in `.data` (occupies file bytes). A non-trivial
> `static std::string g = "hi";` needs a *runtime constructor*, so it lands in
> `.data`/`.bss` for storage **plus** an entry in `.init_array` to run the
> constructor at startup (Part 5.5).

---

## 1.1.6 The stack in detail

The stack is the single most important data structure for understanding calling
conventions, debugging, and unwinding. On x86-64 it grows **downward**: pushing
*decrements* `RSP`.

```
   Higher addresses
   ┌────────────────────────┐
   │  caller's frame        │
   ├────────────────────────┤
   │  argument 7, 8, ...    │ ← args beyond the 6 register args spill here
   ├────────────────────────┤
   │  return address        │ ← pushed by `call`
   ├────────────────────────┤ ◀─ RBP (frame pointer), if used
   │  saved RBP             │
   ├────────────────────────┤
   │  local variables       │
   │  (callee-saved spills) │
   ├────────────────────────┤ ◀─ RSP (top of stack)
   │  red zone (128 bytes)  │ ← scratch below RSP, leaf funcs only (SysV)
   └────────────────────────┘
   Lower addresses
```

Mechanics:

- `call target` pushes the address of the *next* instruction (the return
  address) and jumps to `target`. Equivalent to `push rip_next; jmp target`.
- `ret` pops that address into `RIP`.
- `push reg` = `sub rsp, 8; mov [rsp], reg`.
- `pop reg`  = `mov reg, [rsp]; add rsp, 8`.

> **ABI rule ▸** At the point of a `call`, `RSP` must be 16-byte aligned (per
> the System V ABI). Because `call` pushes 8 bytes, on entry to a function `RSP
> ≡ 8 (mod 16)`. Forgetting this breaks aligned SSE loads (`movaps`) and is a
> classic hand-written-assembly crash.

> **Red zone ▸** The 128 bytes *below* `RSP` are reserved for the current
> (leaf) function to use as scratch without adjusting `RSP`. Signal handlers and
> the kernel must not clobber it. The Windows x64 ABI has **no** red zone.

We dissect frames fully in Part 2.4 (calling conventions) and Part 6.4 (CFI).

---

## 1.1.7 Caches & why layout matters (briefly)

Between the registers and main memory sit caches (L1/L2/L3). They don't change
*correctness*, but they explain why the linker's *ordering* of functions and
data (and features like `--gc-sections`, function reordering, and hot/cold
splitting) affects performance.

```
 fastest, smallest                                 slowest, largest
 ┌────────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────────────┐
 │  Regs  │→  │  L1  │→  │  L2  │→  │  L3  │→  │  Main memory │
 │ ~1 cyc │   │~4 cyc│   │~12cyc│   │~40cyc│   │  ~200+ cyc   │
 └────────┘   └──────┘   └──────┘   └──────┘   └──────────────┘
   B            32-64KiB   256KiB-    several      many GiB
                          1MiB        MiB
```

The linker fetches code in **cache-line** (typically 64-byte) units when the
CPU runs it; keeping hot functions together improves instruction-cache density.
This is why profile-guided optimization and tools like `mold`/`lld` support
section ordering.

---

## 1.1.8 ARM64 (AArch64) in one screen

Because Apple Silicon and cloud ARM are now mainstream, here's the contrast:

```
 x86-64                         AArch64
 ───────                        ───────
 16 GPRs (RAX..R15)             31 GPRs (X0..X30) + XZR/SP
 RIP                            PC (not a GPR)
 RFLAGS                         NZCV condition flags
 variable-length instr (1-15B)  fixed 32-bit instructions
 CISC, mem operands in ALU ops  RISC, load/store architecture
 RSP grows down, push/pop       SP, no push/pop; use stp/ldp pairs
 args: RDI,RSI,RDX,RCX,R8,R9    args: X0..X7
 return: RAX                    return: X0
 call/ret                       bl/ret (ret uses X30 = link register)
```

The biggest conceptual difference: ARM is **load/store** — arithmetic only
operates on registers, and memory is touched only by explicit `ldr`/`str`. The
**link register** `X30` holds the return address instead of it being pushed to
the stack by `call`. ELF and DWARF are identical in structure; only the
relocation types and instruction encodings differ.

---

## Summary

- The CPU is a fetch-decode-execute loop over a flat byte array.
- Registers are the fast working set; sub-register aliasing and the
  zero-extension rule matter for encoding.
- Programs live in a per-process flat **virtual** address space; the linker
  assigns virtual addresses and the loader maps file contents to them.
- The process is divided into `.text`/`.rodata`/`.data`/`.bss`/heap/stack —
  memorize this layout; it underpins all of linking.
- The stack grows down; `call`/`ret` and the 16-byte alignment rule define the
  frame mechanics we'll rely on everywhere.

Next: [1.2 — Bits, bytes, endianness & data representation](02-data-representation.md)
