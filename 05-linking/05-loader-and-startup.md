# 5.5 — The Loader, Program Startup & `_start`

We've built the executable; now we run it. This chapter follows the exact
sequence from `execve` to `main`, through the kernel, the dynamic loader, the C
runtime, and global constructors. Understanding this answers "what runs before
`main`?" precisely.

---

## 5.5.1 The big picture

```
   shell: fork() + execve("./app", argv, envp)
      │
      ▼
   ┌─────────────── KERNEL ─────────────────┐
   │ parse ELF header + program headers     │
   │ mmap PT_LOAD segments                  │
   │ set up stack with argc/argv/envp/auxv  │
   │ find PT_INTERP → load ld.so too        │
   │ transfer control to ld.so's entry      │
   └───────────────┬────────────────────────┘
                   ▼
   ┌──────────── DYNAMIC LOADER (ld.so) ────┐
   │ load DT_NEEDED libraries (recursively) │
   │ apply relocations (.rela.dyn, RELR)    │
   │ run libraries' init (DT_INIT_ARRAY)    │
   │ jump to the executable's e_entry       │
   └───────────────┬────────────────────────┘
                   ▼
   ┌──────────── C RUNTIME (crt) ────────────┐
   │ _start  → __libc_start_main(...)        │
   │   run the exe's .init_array (ctors)     │
   │   call main(argc, argv, envp)           │
   │   call exit(main's return value)        │
   │   run .fini_array (dtors), flush stdio  │
   └─────────────────────────────────────────┘
```

---

## 5.5.2 Kernel phase: from `execve` to first instruction

`execve` replaces the calling process image. The kernel's ELF binary loader
(`fs/binfmt_elf.c`) does:

```
   1. Validate the ELF header (§4.2.5). Reject if not ET_EXEC/ET_DYN.
   2. Read the program header table.
   3. For each PT_LOAD: mmap it at p_vaddr (+ random base if PIE) with p_flags
      permissions, zero-filling the [p_filesz, p_memsz) gap (.bss).
   4. If PT_INTERP present: load the named interpreter (ld.so) similarly,
      at another randomized base.
   5. Build the initial stack: argv, envp, and the auxiliary vector (auxv).
   6. Set the entry: if there's an interpreter, jump to ITS entry; else to the
      executable's e_entry.
```

```
   Initial stack the kernel hands over (top = high addr):
   ┌──────────────────────────┐  high
   │  env strings, argv strs  │
   │  auxv (AT_PHDR, AT_BASE, │
   │   AT_ENTRY, AT_RANDOM..) │  ← key/value pairs: where phdrs are, ld.so base,
   │  NULL                    │     entry, random bytes for stack canary, etc.
   │  envp[] pointers, NULL   │
   │  argv[] pointers, NULL   │
   │  argc                    │  ◀── RSP points here at process entry
   └──────────────────────────┘  low
```

> **The auxiliary vector (auxv) ▸** A kernel→userspace channel of useful values:
> `AT_PHDR` (where the program headers are mapped), `AT_BASE` (ld.so's load
> address), `AT_ENTRY` (the exe's real entry), `AT_RANDOM` (16 random bytes used
> to seed the stack canary), `AT_HWCAP`/`AT_HWCAP2` (CPU feature bits used by
> IFUNC resolvers). View it with `LD_SHOW_AUXV=1 ./app`.

---

## 5.5.3 Dynamic loader phase: `ld.so`

If the binary is dynamic, the kernel gave control to the interpreter. `ld.so`
bootstraps itself (it must relocate *itself* first — a delicate self-hosting
step), then:

```
   1. Read the executable's .dynamic (via PT_DYNAMIC / auxv AT_PHDR).
   2. Build the dependency graph: for each DT_NEEDED, find & mmap the library
      (§5.3.5 search order), recursing into its DT_NEEDEDs. Breadth-first.
   3. Relocation: for every loaded object, apply .rela.dyn (RELATIVE, GLOB_DAT,
      COPY, IRELATIVE) and, if BIND_NOW, .rela.plt (JUMP_SLOT). Process DT_RELR.
   4. Run initializers: for each library (in dependency order, deps first),
      call DT_INIT then DT_INIT_ARRAY entries (the library's constructors).
   5. Transfer control to the executable's entry point (e_entry → _start).
```

```
   Initialization order (so a library's ctors run before its users'):
     leaf deps first → ... → libc → the executable's own .init_array (later, in crt)
```

**Try it ▸** Watch the whole thing:

```bash
LD_DEBUG=all ./demo 2>&1 | less          # exhaustive: search, reloc, init, symbols
LD_DEBUG=statistics ./demo 2>&1 | tail    # relocation counts & timing
strace -e trace=execve,mmap,openat ./demo 2>&1 | head -30   # syscalls
```

---

## 5.5.4 `_start` and `__libc_start_main`

`e_entry` points at `_start` (from `Scrt1.o`/`crt1.o`), *not* `main`. `_start` is
written in assembly and does the minimum to call into libc:

```asm
_start:                          ; the real ELF entry point
    xor   ebp, ebp               ; mark the outermost frame (RBP=0 → unwind stop)
    mov   r9, rdx                ; rtld_fini (dtor registered by ld.so)
    pop   rsi                    ; argc → then argv is at rsp
    mov   rdx, rsp               ; argv
    and   rsp, -16               ; align the stack to 16 bytes (ABI)
    push  rax
    push  rsp
    lea   r8,  [rip + __libc_csu_fini]   ; (older glibc) fini
    lea   rcx, [rip + __libc_csu_init]   ; (older glibc) init
    lea   rdi, [rip + main]              ; main → first arg
    call  __libc_start_main      ; never returns
    hlt
```

`__libc_start_main(main, argc, argv, init, fini, rtld_fini, stack_end)` then:

```
   1. Set up thread-local storage (TLS) for the main thread, the stack guard
      (canary, seeded from AT_RANDOM), and locking.
   2. Register fini/rtld_fini to run at exit (atexit machinery).
   3. Run the executable's constructors: DT_INIT (_init) then DT_INIT_ARRAY[].
   4. Call exit(main(argc, argv, envp)).
   5. exit() runs atexit handlers + DT_FINI_ARRAY (dtors), flushes stdio,
      then _exit (the syscall) to the kernel.
```

```
   So the true call chain is:
   kernel → ld.so → _start → __libc_start_main → [.init_array ctors] → main
                                                   ↓ (return)
                                                 exit → [.fini_array dtors] → _exit
```

---

## 5.5.5 Global constructors: `.init_array` in detail

This is where C++ global/static objects with non-trivial constructors get
initialized **before `main`**.

```c
struct Logger { Logger(){ /* runs before main */ } } g_logger;  // global object
static std::string g_name = compute();   // dynamic init
__attribute__((constructor)) void early(){ /* C-style ctor */ }
```

For each such initializer, the compiler:
1. Emits the actual constructor code as a function.
2. Adds a pointer to a wrapper (that runs the ctor) into the `.init_array`
   section of that TU.

The linker concatenates all TUs' `.init_array` sections into one array; the
runtime walks it in order:

```
   .init_array (after linking):
   ┌──────────────┬──────────────┬──────────────┬─────┐
   │ &ctor_tuA_1  │ &ctor_tuA_2  │ &ctor_tuB_1  │ ... │
   └──────────────┴──────────────┴──────────────┴─────┘
   __libc_csu_init / __libc_start_main walks this array calling each pointer.
```

```
   Ordering guarantees:
   - WITHIN a translation unit: constructors run in source/declaration order.
   - ACROSS translation units: ORDER IS UNSPECIFIED.  ← the "static
     initialization order fiasco"
   - init_priority attribute / linker section ordering can impose some order.
```

> **C++ ▸ Static Initialization Order Fiasco ▸** Because cross-TU `.init_array`
> order is unspecified, a global in `a.cpp` that depends on a global in `b.cpp`
> may be constructed first → uses a not-yet-constructed object → UB. The fix is
> the **Construct On First Use** idiom (function-local static, which is
> initialized lazily and thread-safely via a guard variable — `__cxa_guard_acquire`):
>
> ```cpp
> Foo& instance() { static Foo f; return f; }   // safe, lazy, ordered-by-use
> ```
>
> The `static Foo f` compiles to a guard-variable check (`__cxa_guard_*`) so it's
> initialized exactly once, even with concurrent first calls.

`.fini_array` mirrors this for destructors, run in **reverse** at exit.
`__cxa_atexit` registers destructors of dynamically-initialized objects so they
run in reverse construction order, even across `dlclose`.

**Try it ▸**

```bash
cat > ctor.cpp <<'EOF'
#include <cstdio>
struct T{T(){puts("ctor");} ~T(){puts("dtor");}} g;
int main(){ puts("main"); return 0; }
EOF
g++ ctor.cpp -o ctor && ./ctor      # prints: ctor / main / dtor
readelf -SW ctor | grep init_array  # the section exists
objdump -s -j .init_array ctor      # raw pointers (relocated at load)
```

---

## 5.5.6 The exit path

```
   main returns N
      │
      ▼
   __libc_start_main calls exit(N)
      │  run atexit/__cxa_atexit handlers in LIFO order
      │  run .fini_array (DT_FINI_ARRAY) in reverse  → global dtors
      │  flush & close stdio streams (fflush)
      ▼
   _exit(N)  →  exit_group(2) syscall  →  kernel reaps the process
```

```
   exit(3)   : orderly — runs handlers, flushes buffers. (what main's return does)
   _exit(2)  : immediate — skips handlers/flush. (use after fork before exec)
   abort(3)  : raises SIGABRT — no cleanup, may dump core.
   quick_exit/at_quick_exit : C++11 alternative shutdown path.
```

> **Pitfall ▸** Calling `_exit` (or crashing) skips `.fini_array` and stdio
> flush — buffered `printf` output can be lost, and global destructors don't run.
> Conversely, doing nontrivial work in global destructors that runs at process
> exit can deadlock or use already-freed singletons; many large apps deliberately
> `_exit` to skip destructor chaos.

---

## 5.5.7 Static executables: the simpler path

A fully static binary (`gcc -static`) has **no** `PT_INTERP` and no `ld.so`. The
kernel maps it and jumps straight to `_start`; `__libc_start_main` still runs
`.init_array` and `main`. No dynamic relocation, no GOT/PLT resolution at
runtime (calls are direct). Faster startup, self-contained — at the cost of size
and no `dlopen`/NSS.

```
   dynamic:  kernel → ld.so → _start → ... → main
   static :  kernel ───────▶ _start → ... → main      (ld.so absent)
```

---

## Summary

- Running a program = kernel `execve` (map `PT_LOAD`s, build stack+auxv, load
  `ld.so`) → dynamic loader (load `DT_NEEDED` libs, relocate, run their init) →
  C runtime (`_start` → `__libc_start_main`) → `main`.
- `e_entry` is `_start`, not `main`; `__libc_start_main` sets up TLS/canary,
  runs `.init_array`, calls `main`, then `exit`.
- The **auxv** carries kernel→userspace data (phdr location, ld.so base, random
  canary seed, CPU features).
- Global/static C++ constructors run via `.init_array` before `main` (in-TU
  ordered, cross-TU unordered → the initialization-order fiasco; fix with
  Construct-On-First-Use). Destructors run via `.fini_array`/`__cxa_atexit` in
  reverse at `exit`.
- Static executables skip `ld.so` entirely.

This completes Part 5. → Part 6 turns to how debuggers map all this back to your
source: [6.1 — DWARF overview & sections](../06-dwarf/01-dwarf-overview.md)
