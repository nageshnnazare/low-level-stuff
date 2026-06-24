# 6.4 — Call Frame Information & Stack Unwinding

How does a debugger print a backtrace through optimized code with no frame
pointer? How does a C++ `throw` find the right `catch` four frames up, destroying
locals along the way? Both rely on **Call Frame Information (CFI)** — DWARF's
description of how to undo a function's stack setup. This is the deepest and most
expert-relevant DWARF topic.

---

## 6.4.1 The unwinding problem

To walk up the call stack from a given instruction, you must recover, for each
frame:
- The **return address** (where the caller will resume).
- The **caller's stack pointer** (its CFA — see below).
- The values of **callee-saved registers** the function may have overwritten.

```
   At a crash deep in the call tree, you have only: RIP, RSP, RBP, and memory.
   You must reconstruct:

   frame: leaf()    ← RIP here
          │ who called me? what was RSP/saved regs in my caller?
          ▼
   frame: middle()
          │
          ▼
   frame: main()
          │
          ▼
   __libc_start_main ...
```

With a frame pointer (`RBP` chain) this is easy: each frame saves the previous
`RBP`, forming a linked list. But `-O2` **omits the frame pointer** (§2.4.5),
using `RBP` as a general register. Now there's no chain — you need a *table* that
says, for every PC, how to find the CFA and saved registers. That table is CFI.

---

## 6.4.2 The CFA: Canonical Frame Address

The anchor of unwinding is the **CFA**: by convention on x86-64, the value of
`RSP` *at the call site in the caller* — i.e. just before the `call` that entered
this function, which equals the address one past the return address on the
stack.

```
   ┌────────────────────────┐  higher
   │ caller's frame         │
   ├────────────────────────┤ ◀── CFA  (= RSP in caller right before `call`)
   │ return address         │  CFA-8
   ├────────────────────────┤ ◀── RSP just after `call` (callee entry)
   │ callee's saved regs    │
   │ callee's locals        │
   ├────────────────────────┤ ◀── RSP now (after prologue)
   └────────────────────────┘  lower
```

Everything is expressed *relative to the CFA*, because the CFA is stable for the
whole function even as RSP moves:

```
   return address      = *(CFA - 8)
   caller's RSP        = CFA
   saved RBX (if any)  = *(CFA - 16)     ; "RBX is at CFA-16"
   ...
```

If you know the CFA and the rules for each register, you can reconstruct the
caller's entire register state, then repeat for the next frame up.

---

## 6.4.3 The CFI table (conceptually)

CFI is, conceptually, a giant table: rows = addresses (PCs), columns =
registers, cells = "how to recover this register's caller-value at this PC."

```
   PC range         CFA rule        RA           RBX           RBP
   ──────────       ──────────      ──────       ──────        ──────
   func+0x00        rsp+8           CFA-8        same          same     ; on entry
   func+0x01        rsp+16          CFA-8        same          same     ; after push rbp
   func+0x04        rbp+16          CFA-8        same          CFA-16   ; after mov rbp,rsp
   func+0x10        rbp+16          CFA-8        CFA-24        CFA-16   ; after push rbx
   ...
```

Reading a row: "At this PC, the CFA = (register + offset); the return address is
at CFA-8; RBX's caller-value is saved at CFA-24; RBP's at CFA-16." Just like the
line table (§6.3), this table is too big to store directly, so it's encoded as a
**bytecode program**.

---

## 6.4.4 CIE and FDE: the encoding

CFI lives in `.eh_frame` (runtime) and/or `.debug_frame` (debug). It's a sequence
of two record types:

```
   CIE — Common Information Entry (shared template):
   ┌────────────────────────────────────────────────────────────┐
   │ code_alignment_factor, data_alignment_factor               │
   │ return_address_register   (which reg holds RA; x86-64: 16) │
   │ initial CFI instructions  (the default rules)              │
   │ augmentation ("zR", personality routine ptr, LSDA encoding)│
   └────────────────────────────────────────────────────────────┘

   FDE — Frame Description Entry (one per function):
   ┌────────────────────────────────────────────────────────────┐
   │ pointer to its CIE                                         │
   │ initial_location (function start PC)  + address_range (len)│
   │ CFI instructions (the per-PC rule changes for this func)   │
   │ augmentation data (LSDA pointer for exception handling)    │
   └────────────────────────────────────────────────────────────┘
```

```
   .eh_frame:
   ┌─────┐   ┌─────┐ ┌─────┐ ┌─────┐   ┌─────┐ ┌─────┐ ...
   │ CIE │ ← │ FDE │ │ FDE │ │ FDE │   │ CIE │ │ FDE │
   └─────┘   └──┬──┘ └──┬──┘ └──┬──┘   └─────┘ └─────┘
              each FDE points back to a CIE for shared defaults
```

The CFI instructions (`DW_CFA_*`) build the table:

```
   DW_CFA_def_cfa reg, off        ; CFA = reg + off
   DW_CFA_def_cfa_offset off      ; change just the offset (e.g. after push/sub rsp)
   DW_CFA_def_cfa_register reg    ; change just the register (e.g. switch to rbp)
   DW_CFA_offset reg, n           ; reg is saved at CFA + n*data_align
   DW_CFA_restore reg             ; reg reverts to its initial (CIE) rule
   DW_CFA_advance_loc delta       ; move to a later PC (next table row)
   DW_CFA_nop                     ; padding
```

---

## 6.4.5 Mapping a prologue to CFI

This is the payoff — see the assembly and the CFI side by side:

```asm
 main:
     push rbp                  ; DW_CFA_advance_loc; DW_CFA_def_cfa_offset 16
                               ;   (RSP dropped 8, so CFA = rsp+16)
                               ; DW_CFA_offset rbp, -16  (saved RBP at CFA-16)
     mov  rbp, rsp             ; DW_CFA_advance_loc; DW_CFA_def_cfa_register rbp
                               ;   (now CFA = rbp+16, stable for the body)
     push rbx                  ; DW_CFA_advance_loc; DW_CFA_offset rbx, -24
     sub  rsp, 8
     ...                       ; body — CFA still rbp+16 regardless of RSP
     pop  rbx                  ; DW_CFA_restore rbx
     pop  rbp                  ; DW_CFA_def_cfa rsp, 8
     ret
```

These are exactly the `.cfi_*` directives the compiler/assembler emit (§2.3.4).
You can see them in `gcc -S` output:

```bash
gcc -O2 -S demo.c -o - | grep -E '\.cfi_|main:' | head
```

> **The assembler generates `.eh_frame` ▸** Those `.cfi_startproc`/
> `.cfi_def_cfa_offset`/`.cfi_offset`/`.cfi_endproc` directives in the `.s` are
> how `.eh_frame` gets built. Hand-written assembly that omits them produces
> functions that **can't be unwound through** — backtraces stop, and C++
> exceptions thrown through them call `std::terminate`. Always add CFI to
> hand-written asm that may appear in a C++ call stack.

---

## 6.4.6 Why `.eh_frame` is loaded (the exception connection)

C++ exception handling needs to unwind the stack **at runtime** (no debugger
present), so `.eh_frame` is `SHF_ALLOC` — present in memory in every C++ binary,
even stripped/release ones. The throw path:

```
   throw E;
     │  __cxa_throw allocates the exception object
     ▼
   _Unwind_RaiseException  (libgcc/libunwind)
     │  PHASE 1 (search): walk frames upward using .eh_frame FDEs, asking each
     │     frame's PERSONALITY ROUTINE "do you catch E?" (reads the LSDA)
     ▼  found a handler N frames up?
   PHASE 2 (cleanup): walk again, and for EACH intervening frame run its
     cleanup code (destructors of locals!) via landing pads, then transfer
     control to the matching catch.
```

```
   .eh_frame ─▶ how to unwind each frame (CFA, saved regs, RA)
   .gcc_except_table (LSDA) ─▶ per-function: which PC ranges are in try-blocks,
                               which catch types match, where the landing pads are
   personality routine (__gxx_personality_v0) ─▶ interprets the LSDA to decide
                               "catch here?" and "run these destructors"
```

So three pieces cooperate: **`.eh_frame`** (how to unwind), the **LSDA /
`.gcc_except_table`** (what to do per frame), and the **personality routine**
(the decision logic). This is "zero-cost exceptions": no runtime cost on the
non-throwing path (no setjmp/register), all the cost paid only when actually
throwing, by consulting these tables.

> **C++ ▸ `noexcept` and unwinding ▸** A `noexcept` function that throws calls
> `std::terminate` — the unwinder, during phase 1, hits the noexcept boundary
> and gives up. `-fno-exceptions` omits the LSDA and shrinks `.eh_frame`.
> `-fno-asynchronous-unwind-tables` removes CFI for non-call sites (smaller, but
> breaks `perf`/async profilers' stack walks).

---

## 6.4.7 `.eh_frame_hdr`: making unwinding fast

Searching all FDEs linearly for "which function contains PC X" would be slow.
`.eh_frame_hdr` (pointed to by `PT_GNU_EH_FRAME`) is a **binary-search index**:
a sorted table of (initial_location → FDE pointer), so the unwinder finds the
right FDE in O(log n).

```
   PT_GNU_EH_FRAME ─▶ .eh_frame_hdr
                       ┌───────────────────────────────────────┐
                       │ sorted: [pc0 → fde0][pc1 → fde1] ...  │  binary search
                       └───────────────────────────────────────┘
   _Unwind_Find_FDE(pc): binary-search this table → the FDE for pc.
```

This is also how language runtimes and tools (`backtrace()`,
`libunwind`, profilers) locate unwind info quickly at runtime.

---

## 6.4.8 Inspecting CFI

**Try it ▸**

```bash
# Decoded CFI rules per PC (the table from §6.4.3):
readelf --debug-dump=frames demo
llvm-dwarfdump --debug-frame demo | head -40
objdump --dwarf=frames demo | head -40

# See the compiler's .cfi_ directives that produce it:
g++ -O2 -S exc.cpp -o - | grep -E '\.cfi_|\.section .gcc_except|personality'

# Watch unwinding actually happen:
cat > exc.cpp <<'EOF'
#include <cstdio>
struct Guard{ ~Guard(){ puts("~Guard (unwound)"); } };
void deep(){ Guard g; throw 42; }
int main(){ try { deep(); } catch(int e){ printf("caught %d\n", e); } }
EOF
g++ -g exc.cpp -o exc && ./exc      # prints ~Guard then caught 42
gdb -batch -ex run -ex bt exc 2>/dev/null   # bt walks frames via .eh_frame
```

The destructor running during the throw is phase-2 cleanup driven by the LSDA;
the backtrace working at `-O2` is `.eh_frame` CFI doing its job.

---

## Summary

- **CFI** describes how to undo each function's stack setup so unwinders can
  recover the return address, caller's SP (the **CFA**), and saved callee-saved
  registers — essential when the frame pointer is omitted at `-O2`.
- Everything is expressed relative to the **CFA** (≈ caller's RSP at the call);
  CFI is a per-PC table encoded as a bytecode program (`DW_CFA_*`) in **CIE**
  (shared template) + **FDE** (per-function) records.
- The compiler emits `.cfi_*` directives mirroring the prologue; hand-written asm
  without them can't be unwound (breaks backtraces and C++ exceptions).
- `.eh_frame` is **loaded** (unlike other debug sections) because C++ exceptions
  unwind at runtime; throwing uses `.eh_frame` (how to unwind) + LSDA
  (`.gcc_except_table`, what to do) + the **personality routine** (the
  decision) — "zero-cost" exceptions.
- `.eh_frame_hdr` (`PT_GNU_EH_FRAME`) is a binary-search index for fast FDE
  lookup at runtime.

This completes Part 6. → Part 7 covers advanced cross-cutting topics:
[7.1 — Thread-local storage](../07-advanced/01-thread-local-storage.md)
