# 5.4 — PLT, GOT & Lazy Binding

How does a call to `printf` in your code reach libc's `printf`, when libc is
loaded at a random address chosen at runtime? The answer is the **PLT**
(Procedure Linkage Table) and **GOT** (Global Offset Table). This chapter
dissects both, traces a lazy-bound call instruction by instruction, and explains
the security implications.

---

## 5.4.1 The problem: calling across modules under ASLR

Your code is position-independent and so is libc, each loaded at an
address unknown until runtime. A direct `call 0x<absolute>` is impossible to bake
in. We need an indirection that the dynamic loader can fill in.

```
   your code               libc.so (loaded at random base)
   ─────────               ───────
   call printf ───?───▶    printf at 0x7f....3210   (address known only at runtime)

   Solution: call a local stub that jumps through a writable pointer slot the
   loader fills in.  →  the PLT (stub) + GOT (pointer slot).
```

Two requirements drive the design:
1. **Code stays read-only and shareable** (no runtime patching of `.text`).
2. **The actual address is resolved once**, ideally lazily (only if called).

The GOT (writable data) holds the pointers; the PLT (read-only code) is the
trampoline that reads them.

---

## 5.4.2 The two tables

```
   GOT — Global Offset Table  (.got / .got.plt, in a WRITABLE segment)
   ───────────────────────────────────────────────────────────────
   An array of 8-byte pointer slots. The dynamic loader writes the resolved
   addresses of external functions and data here. Code reads through it.

   PLT — Procedure Linkage Table  (.plt, in the EXECUTABLE/read-only segment)
   ───────────────────────────────────────────────────────────────
   An array of tiny code stubs, one per imported function. Each stub jumps to
   the address stored in that function's GOT slot.
```

```
   call printf  ─▶  printf@plt (stub) ─▶ jmp *[GOT slot for printf] ─▶ real printf
                    (read-only code)      (writable pointer slot)       (in libc)
```

Why split GOT into `.got` and `.got.plt`?

```
   .got      : slots for data symbols + GOT[0..2] bookkeeping; can be made
               read-only after startup (RELRO).
   .got.plt  : slots for function symbols resolved via the PLT; must stay
               writable if LAZY binding is used (the resolver writes them on
               first call). With FULL RELRO these are resolved at startup and
               then made read-only too.
```

---

## 5.4.3 GOT[0..2]: the lazy-binding bootstrap

The first three entries of `.got.plt` are special, set up so the PLT can call
back into the loader:

```
   .got.plt:
   ┌──────────────┬──────────────────────────────────────────────────┐
   │ GOT[0]       │ address of the .dynamic section (for ld.so)      │
   │ GOT[1]       │ link_map* — identifies THIS module to ld.so      │
   │ GOT[2]       │ address of _dl_runtime_resolve (the resolver)    │
   ├──────────────┼──────────────────────────────────────────────────┤
   │ GOT[3]       │ slot for function #0   (printf)                  │
   │ GOT[4]       │ slot for function #1   (puts)                    │
   │ ...          │ ...                                              │
   └──────────────┴──────────────────────────────────────────────────┘
   GOT[1] and GOT[2] are filled by ld.so at startup; the per-function slots
   initially point BACK into their PLT stub (to trigger resolution on first use).
```

---

## 5.4.4 Anatomy of a (traditional) PLT entry

```
   PLT[0]  (the common resolver trampoline):
       push  [GOT+8]            ; push link_map (GOT[1])
       jmp   *[GOT+16]          ; jump to _dl_runtime_resolve (GOT[2])

   PLT[n]  (one per imported function, e.g. printf):
   printf@plt:
       jmp   *[GOT_slot_n]      ; (1) jump through printf's GOT slot
       push  $n                 ; (2) reloc index (only reached on first call)
       jmp   PLT[0]             ; (3) go to the common resolver
```

```
   The clever part: the GOT slot INITIALLY points at instruction (2) of its own
   PLT entry. So the first `jmp *[GOT_slot]` falls through to (2)+(3), invoking
   the resolver. The resolver overwrites the GOT slot with the REAL address.
   Every subsequent call's `jmp *[GOT_slot]` goes straight to the real function.
```

---

## 5.4.5 Tracing a lazy-bound call, step by step

```
   FIRST call to printf:
   ─────────────────────
   (A) your code:   call printf@plt
   (B) printf@plt:  jmp *[GOT_slot]
                       │  GOT_slot still points to (C) below (not resolved yet)
                       ▼
   (C) printf@plt+6: push $reloc_index
   (D)               jmp PLT[0]
   (E) PLT[0]:       push link_map ; jmp *_dl_runtime_resolve
                       │
                       ▼
   (F) _dl_runtime_resolve(link_map, reloc_index):
          - looks up "printf" in libc's .dynsym via .gnu.hash
          - finds real address, e.g. 0x7f...3210
          - WRITES it into GOT_slot   ◀── the one-time fixup
          - tail-calls the real printf  (so this call also works)

   SECOND and later calls:
   ───────────────────────
   (A) call printf@plt
   (B) printf@plt: jmp *[GOT_slot]   ── GOT_slot now = 0x7f...3210
                       └────────────────▶ real printf directly. Done.
```

```
   GOT_slot lifecycle:
     before first call:  → printf@plt+6  (triggers resolver)
     after first call:   → 0x7f...3210   (real printf in libc)
```

This is **lazy binding**: the cost of symbol resolution is paid only for
functions actually called, and only once each. The corresponding relocation is
`R_X86_64_JUMP_SLOT` in `.rela.plt`.

**Try it ▸** Watch the GOT slot change:

```bash
printf '#include <stdio.h>\nint main(){puts("a");puts("b");return 0;}\n' > p.c
gcc -no-pie -O0 p.c -o p           # -no-pie keeps addresses simple to read
objdump -d -j .plt p | head -30    # the PLT stubs
objdump -R p                       # dynamic relocations: JUMP_SLOT for puts
# In gdb: break main; then `x/2i 0x<puts@plt>`; inspect the GOT slot before and
# after the first puts call to see it get patched.
```

---

## 5.4.6 The modern `.plt.sec` / IBT variant

On recent toolchains with Intel CET (Control-flow Enforcement Technology), the
PLT is split for security:

```
   .plt      : minimal stubs ending in jmp to .plt.sec
   .plt.sec  : the actual indirect jumps, each starting with `endbr64`
               (a landing pad required by IBT for indirect branch targets)
```

`endbr64` marks a legal indirect-jump target; without it, an indirect jump
faults — defeating many ROP/JOP exploits. You'll see `endbr64` as the first
instruction of nearly every function and PLT entry on CET-enabled builds.

---

## 5.4.7 Eager binding: BIND_NOW and Full RELRO

Lazy binding leaves `.got.plt` writable (the resolver patches it at runtime),
which is an attacker target (overwrite a GOT slot → hijack control flow). The
hardened alternative resolves everything at startup and locks the GOT:

```
   -Wl,-z,now      → DF_1_NOW / DT_BIND_NOW: resolve all PLT relocs at load
   -Wl,-z,relro    → PT_GNU_RELRO: mark .got etc. read-only AFTER relocation
   together = "Full RELRO" (the common hardened default on many distros)
```

```
   LAZY (default-ish):     GOT.plt writable for life   → fast start, weaker
   BIND_NOW + Full RELRO:  resolve all up front, then mprotect GOT read-only
                           → slower start, GOT can't be overwritten
```

```
   startup with BIND_NOW:
     ld.so resolves EVERY JUMP_SLOT immediately (no lazy stubs needed)
     then ld.so mprotect()s the RELRO region to read-only
     → an attacker who gains a write primitive can't tamper with the GOT
```

Check a binary's posture:

```bash
readelf -d demo | grep -E 'BIND_NOW|FLAGS|RELRO'
readelf -l demo | grep RELRO
# tools: `checksec --file=demo`  →  reports "Full RELRO" / "Partial" / "No RELRO"
```

> **Performance vs security ▸** Lazy binding optimizes startup of programs that
> call only a fraction of their imports. BIND_NOW pays it all upfront but enables
> Full RELRO and is increasingly the default for security. For latency-sensitive
> short-lived processes the trade can matter.

---

## 5.4.8 GOT for data: `R_X86_64_GLOB_DAT`

Imported *variables* (not functions) also go through the GOT, but without a PLT
stub — the code loads the address from a `.got` slot directly:

```
   extern int errno_like;     // defined in another module
   mov rax, [rip + errno_like@GOTPCREL]   ; load the GOT slot → address
   mov eax, [rax]                         ; deref to read the value
```

The slot is filled by an `R_X86_64_GLOB_DAT` dynamic relocation at load time.
This is why imported data access is a double indirection (GOT slot → variable),
and why `-fvisibility=hidden`/protected (avoiding the GOT for internal symbols)
speeds things up.

> **Copy relocations ▸** For backward compatibility with non-PIC executables,
> imported *data* could instead use `R_X86_64_COPY`: the variable's storage is
> allocated in the executable's `.bss` and the loader *copies* the library's
> initial value there, so the exe references a fixed local address. This is the
> legacy mechanism and a reason `-fPIC` libraries are preferred. PIE + GOT
> avoids copy relocs.

---

## Summary

- The **GOT** (writable pointer slots) + **PLT** (read-only trampoline stubs)
  let position-independent code call/access symbols in modules loaded at runtime
  addresses, without patching `.text`.
- `GOT[0..2]` bootstrap the loader's lazy resolver; per-function slots start by
  pointing back into their PLT stub to trigger resolution on first call.
- **Lazy binding** resolves each function once, on first use, via
  `_dl_runtime_resolve`, patching the GOT slot (`R_X86_64_JUMP_SLOT`).
- **BIND_NOW + RELRO** resolve everything at startup and re-protect the GOT
  read-only — slower start, stronger security (defeats GOT-overwrite attacks).
- Imported data uses GOT slots filled by `GLOB_DAT` (or legacy `COPY`
  relocations); CET adds `endbr64` landing pads and a split `.plt.sec`.

Next: [5.5 — The loader, program startup & `_start`](05-loader-and-startup.md)
