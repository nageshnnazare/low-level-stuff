# 7.4 — Security Mitigations (RELRO, PIE, Canaries, CET, …)

Modern toolchains weave a layer of exploit mitigations into the binary at compile
and link time. Each one is implemented through the ELF/linker/loader machinery
you've now learned. This chapter explains what they are, how they work at the
binary level, and how to inspect them.

---

## 7.4.1 The threat model in one diagram

```
   Classic memory-corruption exploit chain:
   ┌──────────────┐   ┌─────────────────┐   ┌──────────────────────┐
   │ memory bug   │──▶│ overwrite a     │──▶│ redirect control     │
   │ (overflow,   │   │ code pointer    │   │ flow to attacker code│
   │  UAF, ...)   │   │ (ret addr, GOT, │   │ (shellcode / ROP)    │
   │              │   │  vtable, fnptr) │   │                      │
   └──────────────┘   └─────────────────┘   └──────────────────────┘

   Mitigations break links in this chain:
     - NX/W^X      → can't execute injected data
     - ASLR/PIE    → can't predict addresses
     - canaries    → detect stack ret-addr overwrite
     - RELRO       → can't overwrite the GOT
     - CET/CFI     → can't divert indirect branches/returns
     - Fortify     → catch overflows in libc calls
```

---

## 7.4.2 NX / W^X — non-executable data

The oldest mitigation: memory is either writable **or** executable, never both
(W xor X). Implemented via page permissions in the `PT_LOAD` segments and the
`PT_GNU_STACK` marker (§4.4.3).

```
   .text   → r-x   (execute, not write)
   .data   → rw-   (write, not execute)
   stack   → rw-   (PT_GNU_STACK without PF_X)
   heap    → rw-

   → injected shellcode in a writable buffer can't be executed (page faults).
```

This forced attackers to move from code injection to **code reuse** (ROP/JOP),
which the later mitigations target.

```bash
readelf -l app | grep -A1 GNU_STACK    # RW (good) vs RWE (executable stack, bad)
```

> **Pitfall ▸** Linking even one object lacking a `.note.GNU-stack` marker can
> flip the whole binary to an executable stack. Hand-written `.s` should include
> `.section .note.GNU-stack,"",@progbits`. Linkers now warn (`missing
> .note.GNU-stack section`).

---

## 7.4.3 ASLR & PIE — unpredictable addresses

**ASLR** (Address Space Layout Randomization) is a kernel feature that loads
segments at randomized base addresses each run, so attackers can't hardcode
target addresses. But the *executable itself* is only randomized if it's a
**PIE** (Position-Independent Executable, `ET_DYN`).

```
   non-PIE (ET_EXEC):  .text always at 0x400000  → predictable → easy to attack
   PIE (ET_DYN):       base = 0x5xxxxxxxxxxx (random each run) → unpredictable

   run 1:  ./app   main @ 0x55a3c1e2b149
   run 2:  ./app   main @ 0x5612f8d44149     ← different base
```

PIE requires position-independent code (`-fPIE`/`-fPIC`) so all internal
references are RIP-relative or GOT-based (Parts 2.5, 5.4), letting the loader
place the image anywhere. The cost: the GOT indirections and the
`R_X86_64_RELATIVE` startup fixups (§4.6.5).

```bash
cat /proc/sys/kernel/randomize_va_space     # 2 = full ASLR
gcc -fPIE -pie app.c -o pie_app             # PIE (modern default on most distros)
gcc -fno-pie -no-pie app.c -o nopie_app     # opt out
file pie_app                                # "pie executable" / ET_DYN
# verify randomization:
./pie_app & p=$!; cat /proc/$p/maps | head -1; kill $p   # run twice → base differs
```

---

## 7.4.4 Stack canaries — detecting return-address smashing

A **canary** (a.k.a. stack cookie) is a random value placed between local buffers
and the saved return address. The function checks it before returning; if a
buffer overflow corrupted the return address, it also corrupted the canary, and
the check aborts.

```
   stack frame with a canary:
   ┌────────────────────┐  high
   │ return address     │
   │ saved RBP          │
   ├────────────────────┤
   │ CANARY (random)    │  ◀── inserted by -fstack-protector
   ├────────────────────┤
   │ char buf[64]       │  ◀── overflow here grows UP toward the canary/retaddr
   └────────────────────┘  low

   prologue:  mov rax, %fs:0x28 ; mov [rbp-8], rax   ; load canary from TLS, store
   epilogue:  mov rax, [rbp-8] ; xor rax, %fs:0x28
              jne __stack_chk_fail                    ; mismatch → abort
```

The canary value lives in TLS (`%fs:0x28`), seeded at startup from `AT_RANDOM`
(§5.5.2). Levels:

```
   -fstack-protector          : protect functions with char buffers (heuristic)
   -fstack-protector-strong   : broader heuristic (recommended default)
   -fstack-protector-all      : every function (costly)
   -fstack-protector-explicit : only __attribute__((stack_protect)) functions
```

```bash
gcc -fstack-protector-strong -S app.c -o - | grep -E 'stack_chk|fs:0x28'
```

> **Limit ▸** Canaries protect *return addresses* against *linear* overflows. They
> don't stop targeted writes (out-of-bounds index), GOT overwrites, or heap
> corruption. Hence the other layers.

---

## 7.4.5 RELRO — locking the GOT

As covered in §5.4.7: the GOT is a table of code pointers in writable memory — a
prime overwrite target ("GOT hijacking"). **RELRO** (RELocation Read-Only) has
the loader re-protect it after relocations finish.

```
   Partial RELRO (-Wl,-z,relro):
     .got, .dynamic, .init_array → made read-only after startup.
     BUT .got.plt stays writable (lazy binding still patches it).

   Full RELRO (-Wl,-z,relro,-z,now):
     also forces BIND_NOW → resolve ALL PLT entries at startup,
     then make .got.plt read-only too. Nothing left writable to hijack.
```

```
   timeline (Full RELRO):
     ld.so applies all relocations (incl. JUMP_SLOTs) → mprotect(RELRO region, RO)
     from then on, any write to the GOT faults.
```

```bash
readelf -l app | grep RELRO
readelf -d app | grep -E 'BIND_NOW|FLAGS'
checksec --file=app           # reports No / Partial / Full RELRO  (if installed)
```

Trade-off: Full RELRO slows startup (eager binding, §5.4.7) but is the hardened
default on most distributions.

---

## 7.4.6 CET — hardware control-flow integrity

Intel **CET** (and ARM's **BTI**/**PAC**) add hardware enforcement against
control-flow hijacking, integrated via ELF **GNU properties**
(`PT_GNU_PROPERTY`, `.note.gnu.property`):

```
   IBT (Indirect Branch Tracking):
     every valid indirect-branch target must begin with `endbr64`.
     An indirect jmp/call to an address without endbr64 → #CP fault.
     → defeats JOP/COP (jump/call-oriented programming).

   Shadow Stack (SHSTK):
     a separate, protected stack holds a copy of return addresses. `ret`
     compares the normal return address with the shadow copy; mismatch → fault.
     → defeats ROP (return-oriented programming) and ret-addr overwrite.
```

```
   every function / PLT entry on a CET build starts with:
       endbr64                  ; legal indirect-branch landing pad
       ...
```

Enable with `-fcf-protection=full`. The linker only marks the output CET-enabled
if **all** input objects opt in (the GNU property is AND-merged), so one
non-CET object disables it — a common gotcha.

```bash
gcc -fcf-protection=full -c app.c && readelf -n app.o | grep -A3 properties
objdump -d app | grep endbr64 | head      # the landing pads
```

ARM analogues: **BTI** (`bti` landing-pad instruction, like IBT) and **PAC**
(Pointer Authentication — cryptographically signs return addresses in the LR;
ubiquitous on Apple Silicon).

---

## 7.4.7 FORTIFY_SOURCE & other compile-time checks

```
   -D_FORTIFY_SOURCE=2/3   : replace memcpy/strcpy/sprintf/... with checked
                             variants (__memcpy_chk) that know the dest buffer
                             size at compile time and abort on overflow.
   -Wl,-z,noexecstack      : force non-executable stack.
   -Wl,-z,separate-code    : keep code and data on separate pages (no mixing).
   -fsanitize=address/undefined : runtime instrumentation (debug/CI, not prod).
   -ftrivial-auto-var-init=zero : auto-zero locals (kills uninitialized reads).
```

`__memcpy_chk` and friends are why a `memcpy` of a known-too-large size can
become a link-time/runtime abort instead of silent corruption.

---

## 7.4.8 The hardening checklist

```
 ┌──────────────────────┬───────────────────────────────────────────────────────┐
 │ Mitigation           │ How to enable                                         │
 ├──────────────────────┼───────────────────────────────────────────────────────┤
 │ PIE + ASLR           │ -fPIE -pie  (usually default)                         │
 │ NX / non-exec stack  │ -Wl,-z,noexecstack  (+ .note.GNU-stack in asm)        │
 │ Full RELRO           │ -Wl,-z,relro,-z,now                                   │
 │ Stack canaries       │ -fstack-protector-strong                              │
 │ Fortify              │ -D_FORTIFY_SOURCE=3 -O2                               │
 │ CET (x86) / BTI(ARM) │ -fcf-protection=full  /  -mbranch-protection=standard │
 │ Separate code        │ -Wl,-z,separate-code                                  │
 │ No exec stack warns  │ keep .note.gnu.property AND-merged across all objects │
 └──────────────────────┴───────────────────────────────────────────────────────┘
```

**Try it ▸** Audit any binary:

```bash
# If checksec is available (apt install checksec):
checksec --file=/bin/ls

# Manual equivalent:
readelf -hld /bin/ls | grep -E 'Type:|GNU_STACK|RELRO|BIND_NOW|FLAGS'
readelf -n /bin/ls | grep -iE 'properties|SHSTK|IBT'
objdump -d /bin/ls | grep -c endbr64
```

---

## Summary

- Every modern mitigation is built on the ELF/linker/loader mechanics from this
  guide: page permissions (NX/W^X), `ET_DYN`+RIP-relative code (PIE/ASLR), TLS
  canary slots (`%fs:0x28` from `AT_RANDOM`), GOT re-protection (RELRO +
  BIND_NOW), and GNU properties (CET/BTI/PAC).
- They break different links in the exploit chain: NX stops code injection;
  ASLR/PIE remove predictable addresses; canaries detect return-address smashing;
  RELRO blocks GOT hijacking; CET/shadow-stack defeat ROP/JOP.
- Mitigations are *composed* — defense in depth — and several require **all**
  input objects to cooperate (CET property merge, `.note.GNU-stack`).
- Use `checksec`/`readelf -hlnd` to audit, and the hardening checklist to build
  defensively (with the usual startup-time/size trade-offs).

Next: [7.5 — The binary-analysis toolbox](05-tooling-and-debugging.md)
