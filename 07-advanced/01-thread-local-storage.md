# 7.1 — Thread-Local Storage (TLS)

`thread_local int x;` gives every thread its own copy of `x`. Implementing this
efficiently requires cooperation between the compiler, assembler, linker, dynamic
loader, and the threading runtime — and a clever set of relocations and access
models. This chapter dissects it.

---

## 7.1.1 The problem

A normal global has one instance shared by all threads. A thread-local has **N
instances**, one per thread, each accessed by the *same* source name but
resolving to *different* memory per thread.

```
   thread_local int counter;

   Thread A: counter ──▶ [A's TLS block]+offset
   Thread B: counter ──▶ [B's TLS block]+offset
   Thread C: counter ──▶ [C's TLS block]+offset
                  same name, same offset, different blocks
```

The trick: each thread has a private **Thread Control Block (TCB)** reachable via
a dedicated register, and thread-locals are addressed as *offsets* from it.

---

## 7.1.2 The thread pointer: FS/GS

x86-64 repurposes the otherwise-vestigial **`FS`** segment register as the
**thread pointer** on Linux (Windows/macOS use `GS`). `FS` points at each
thread's TCB. The compiler emits `%fs:`-prefixed accesses.

```
   %fs ─────────────▶ ┌──────────────────────┐  ← Thread Control Block (per thread)
                      │ TCB header (self ptr,│
                      │  dtv pointer, ...)   │
                      ├──────────────────────┤
                      │ .tdata copy          │  ← initialized thread-locals
                      │ .tbss  (zeroed)      │  ← zero-init thread-locals
                      └──────────────────────┘

   access:  mov eax, %fs:offset       ; read a thread-local at fixed offset
```

```
   On AArch64 the thread pointer is the TPIDR_EL0 system register, read with
   `mrs x0, tpidr_el0`. Same concept, different mechanism.
```

The kernel sets `FS_BASE` (via `arch_prctl`/`set_thread_area`) per thread; the
threading library (glibc/pthreads) allocates and initializes each TCB.

---

## 7.1.3 The TLS template: `.tdata` and `.tbss`

The *initial values* of thread-locals form a **template** that each new thread
copies. This template is in two sections, grouped by `PT_TLS`:

```
   .tdata  (SHF_ALLOC|WRITE|TLS, PROGBITS) : initialized TLS, e.g. thread_local int x=5;
   .tbss   (SHF_ALLOC|WRITE|TLS, NOBITS)   : zero-init TLS,  e.g. thread_local int y;

   PT_TLS segment describes the template:
     p_filesz = size of .tdata (bytes to copy)
     p_memsz  = .tdata + .tbss (total per-thread TLS size)
     p_align  = TLS alignment

   thread creation:  new TCB ; copy .tdata template ; zero the .tbss part
```

> **Per-thread cost ▸** Every thread allocates `p_memsz` bytes of TLS at
> creation and memcpy's the `.tdata` template. Large thread-locals (e.g. a big
> `thread_local std::array`) multiply across thousands of threads — a real
> memory and thread-spawn-latency cost in server software.

---

## 7.1.4 The four TLS access models

The compiler picks an access *model* based on optimization flags and whether the
variable is in the main executable or a shared library, trading speed for
flexibility. From fastest/least-flexible to slowest/most-flexible:

```
 ┌──────────────────────────┬───────────────────────────────────────────────────┐
 │ Model                    │ When usable / how it works                        │
 ├──────────────────────────┼───────────────────────────────────────────────────┤
 │ 1. Local Exec (LE)       │ var defined in the MAIN executable. Offset from FS│
 │                          │ is a link-time constant. FASTEST: one mov.        │
 │ 2. Initial Exec (IE)     │ var in a lib loaded at startup (not dlopen'd).    │
 │                          │ Offset loaded from a GOT slot (one extra load).   │
 │ 3. Local Dynamic (LD)    │ multiple thread-locals in the same dlopen'able    │
 │                          │ module; one __tls_get_addr call amortized.        │
 │ 4. General Dynamic (GD)  │ most general: var in any dlopen'able module.      │
 │                          │ Calls __tls_get_addr(module,offset). SLOWEST.     │
 └──────────────────────────┴───────────────────────────────────────────────────┘
```

```
   Local Exec (the fast path):
     mov  eax, %fs:0xfffffffffffffffc      ; var at FS + (constant negative offset)
     → one instruction, offset baked in at link time.  Reloc: R_X86_64_TPOFF32

   General Dynamic (the general path):
     lea  rdi, var@TLSGD[rip]              ; build a (module, offset) descriptor
     call __tls_get_addr@PLT               ; runtime call returns &var for THIS thread
     → returns the address; works even for dlopen'd modules.
       Relocs: R_X86_64_TLSGD, R_X86_64_DTPMOD64, R_X86_64_DTPOFF64
```

Select with `-ftls-model=local-exec|initial-exec|local-dynamic|global-dynamic`.
The linker can also **relax** GD→IE→LE when it discovers the variable is closer
than the compiler assumed (e.g. statically linked) — a TLS-specific optimization.

---

## 7.1.5 The DTV and `__tls_get_addr` (dynamic models)

For dynamically-loaded modules, the offset from `FS` isn't known at compile time
(the module might be `dlopen`ed later). The runtime uses a **Dynamic Thread
Vector (DTV)**: a per-thread array mapping *module id* → that module's TLS block
for this thread.

```
   per thread:
     FS ─▶ TCB ─▶ DTV ─┬─▶ [module 1's TLS block]
                       ├─▶ [module 2's TLS block]
                       └─▶ [module 3's TLS block (dlopen'd, lazily allocated)]

   __tls_get_addr(GOT_entry{module_id, offset}):
       block = thread.dtv[module_id]      ; find this thread's block for the module
       if block not yet allocated: allocate & init it (lazy, for dlopen'd modules)
       return block + offset
```

The `DTPMOD64` relocation fills the module id, `DTPOFF64` the offset; both live
in a GOT-resident descriptor the call consumes. This indirection is why
General-Dynamic TLS is the slowest model — a function call plus possible lazy
allocation per access (though caching helps).

---

## 7.1.6 The relocation zoo for TLS

```
 ┌────────────────────────┬───────────────────────────────────────────────────┐
 │ R_X86_64_TPOFF32       │ Local Exec: 32-bit offset from thread pointer     │
 │ R_X86_64_GOTTPOFF      │ Initial Exec: GOT slot holding TP offset          │
 │ R_X86_64_TLSGD         │ General Dynamic: descriptor for __tls_get_addr    │
 │ R_X86_64_TLSLD         │ Local Dynamic: module descriptor                  │
 │ R_X86_64_DTPMOD64      │ (runtime) module id in a DTV descriptor           │
 │ R_X86_64_DTPOFF64/32   │ (runtime) offset within the module's TLS block    │
 │ R_X86_64_TPOFF64       │ (runtime) offset from thread pointer              │
 └────────────────────────┴───────────────────────────────────────────────────┘
```

**Try it ▸** Watch the model change with flags:

```bash
cat > tls.c <<'EOF'
__thread int counter;          /* or C++ thread_local */
int bump(void){ return ++counter; }
EOF
gcc -O2 -S tls.c -o - | grep -A4 bump                 # default model (PIC → GD/IE)
gcc -O2 -ftls-model=local-exec -S tls.c -o - | grep -A4 bump   # LE: single %fs mov
gcc -O2 -c tls.c -o tls.o && readelf -rW tls.o        # the TLS relocations
```

---

## 7.1.7 C++ specifics

```cpp
   thread_local int n = 0;                  // POD: trivial, lives in .tdata/.tbss
   thread_local std::string s = "hi";       // non-trivial: needs per-thread CTOR
```

For non-trivially-constructed thread-locals, the compiler must run the
constructor **the first time each thread touches the variable** (and register a
destructor for thread exit). This is implemented with a guard and a wrapper:

```
   access to `s` →  call __tls_init_wrapper:
                      if (!this_thread_initialized_s) {
                          construct s in this thread's TLS block;
                          __cxa_thread_atexit(dtor, &s, dso_handle);  // run at thread exit
                          mark initialized;
                      }
                      return &s;
```

> **Pitfalls ▸**
> - **Initial-Exec in a dlopen'd library** can fail at load: IE assumes a TLS
>   slot reserved at startup, but a `dlopen`ed module has none → `cannot allocate
>   memory in static TLS block`. Fix: build the library with global-dynamic
>   (`-ftls-model=global-dynamic`, the default for `-fPIC`) or reserve TLS via
>   `glibc.rtld.optional_static_tls`.
> - **Non-trivial `thread_local` is not free**: every access may check the guard;
>   destructors run at thread exit via `__cxa_thread_atexit`.
> - Taking the address of a thread-local and passing it to another thread is a
>   dangling-pointer bug — the storage is private and freed at thread exit.

---

## Summary

- TLS gives each thread a private copy of a variable, addressed as an offset from
  a per-thread control block reached via `FS` (x86-64 Linux) / `GS` / `TPIDR_EL0`
  (AArch64).
- The initial values are a template in `.tdata` (PROGBITS) + `.tbss` (NOBITS),
  described by `PT_TLS`; each new thread copies/zeros it.
- Four **access models** (Local Exec → Initial Exec → Local Dynamic → General
  Dynamic) trade speed for the ability to handle `dlopen`'d modules; the linker
  can relax to faster models. Each has its own relocation types.
- Dynamic models use a per-thread **DTV** and `__tls_get_addr` for lazy
  per-module TLS allocation.
- C++ non-trivial `thread_local` adds per-thread lazy construction (guarded) and
  `__cxa_thread_atexit` destruction; Initial-Exec TLS in `dlopen`'d libs is a
  classic failure.

Next: [7.2 — Link-time optimization (LTO)](02-link-time-optimization.md)
