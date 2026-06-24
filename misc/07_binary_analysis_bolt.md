# Binary Analysis, Instrumentation, and BOLT (Binary Optimization and Layout Tool)

This document is a **deep, tutorial-style survey** of **binary analysis**, **instrumentation**, **dynamic binary instrumentation (DBI)**, **binary rewriting**, and **BOLT**—the **Binary Optimization and Layout Tool** developed at Meta (formerly Facebook) and integrated into the **LLVM** project. The emphasis is on **how these systems work**, **how they relate**, and **how to use BOLT in practice** with realistic profiling and measurement workflows.

---

## Table of contents

1. [Binary analysis and instrumentation overview](#1-binary-analysis-and-instrumentation-overview)
2. [Static binary analysis tools](#2-static-binary-analysis-tools)
3. [Dynamic binary instrumentation (DBI)](#3-dynamic-binary-instrumentation-dbi)
4. [Binary rewriting](#4-binary-rewriting)
5. [BOLT in depth](#5-bolt-in-depth)
6. [How BOLT works](#6-how-bolt-works)
7. [Using BOLT practically](#7-using-bolt-practically)
8. [Profile collection](#8-profile-collection)
9. [BOLT optimizations explained](#9-bolt-optimizations-explained)
10. [BOLT in production](#10-bolt-in-production)
11. [Other binary optimization tools](#11-other-binary-optimization-tools)
12. [Practical examples](#12-practical-examples)

---

## 1. Binary analysis and instrumentation overview

### 1.1 What binary analysis is

**Binary analysis** is the study and manipulation of **machine code** and **program images** (executables, shared libraries, firmware blobs, kernel modules) **without requiring the original source code**—though debug information (DWARF, PDB) can radically improve precision when present.

Goals include:

- **Understanding behavior**: what instructions run, how control flows, what data is accessed.
- **Finding bugs and vulnerabilities**: memory safety issues, logic flaws, exploitable patterns.
- **Performance work**: hot paths, cache behavior, branch prediction effectiveness, layout inefficiencies.
- **Compatibility and migration**: reverse engineering proprietary interfaces or old ABIs.
- **Hardening and transformation**: rewriting binaries for CFI, sandboxing, or optimization.

Binary analysis sits at the intersection of **compiler backends**, **operating systems** (loaders, memory management), **computer architecture** (pipelines, caches, predictors), and **formal or heuristic program reasoning**.

### 1.2 Static vs dynamic analysis

| Dimension | Static analysis | Dynamic analysis |
|-----------|-----------------|------------------|
| **When it runs** | Before or without execution (offline on disk) | While the program runs (online) |
| **Concrete state** | Reasons over **all potential** paths (approximate) | Observes **one or many concrete** executions |
| **Strengths** | Can cover rare paths; no need for inputs; scales to batch scanning | Ground truth for **what actually executed**; handles dynamic code generation |
| **Weaknesses** | Undecidability, abstraction loss (pointers, indirect branches), may over-approximate | Path coverage depends on workload; nondeterminism; overhead |
| **Representative artifacts** | Disassembly, CFG, call graphs, type/shape recovery | Traces, profiles, coverage maps, sanitizer reports |

**Conceptual picture: two lenses on the same program**

```
                    +---------------------------+
                    | Program binary on disk    |
                    | (.text, .data, metadata)  |
                    +---------------------------+
                           |            |
           static study    |            |   dynamic study
                           v            v
              +----------------+    +------------------+
              | All plausible  |    | Observed traces  |
              | paths (model)  |    | & measurements   |
              +----------------+    +------------------+
                           \              /
                            \            /
                             v          v
                           Combined insight
                    (coverage + soundness tradeoffs)
```

### 1.3 Binary instrumentation categories

At a high level, **instrumentation** means **inserting extra instructions or callbacks** so you can observe or change behavior. For binaries, common categories:

1. **Compile-time instrumentation**  
   The compiler emits checks or counters (e.g. ASan, fuzzing instrumentation, PGO counters in the IR). Requires rebuild from source.

2. **Link-time / post-link instrumentation**  
   The linker or a post-link tool inserts stubs (e.g. some sanitizer modes, LTO-visible counters). Still needs participation of the toolchain.

3. **Static binary instrumentation / rewriting**  
   The **finished** `.text` is modified on disk or during loading: insert trampolines, replace instruction sequences, add new sections. Examples: **BOLT** (optimization layout), **e9patch**, some security tools.

4. **Dynamic binary instrumentation (DBI)**  
   A **runtime engine** copies or JIT-compiles basic blocks, **inserting probes** in a code cache. The original binary may be read-only; execution runs mostly from the **cache**. Examples: **Pin**, **DynamoRIO**, **Valgrind**, **Frida** (hybrid: often injecting a JS engine + code patching).

5. **Hardware-assisted observation**  
   PMU sampling, **LBR**, **Intel PT**, hardware breakpoints—**no instruction insertion**, but strong **temporal** behavior signals for profiling.

### 1.4 Taxonomy of binary analysis tools (ASCII)

```
                         Binary Analysis & Instrumentation
                                       |
        +------------------------------+------------------------------+
        |                              |                              |
   OFFLINE                           ONLINE                    HARDWARE-ASSISTED
   (no run /                        (runtime)                   (PMU / trace)
    single pass)                         |
        |                    +-----------+-----------+
        |                    |                       |
        v                    v                       v
   +-----------+        +------------+          +--------------+
   | Static    |        | DBI        |          | Sampling /   |
   | disasm /  |        | (Pin,      |          | tracing      |
   | decompile |        |  Dynamo,   |          | (perf, PT,   |
   | / SMT     |        |  Valgrind) |          |  LBR, etc.)  |
   +-----------+        +------------+          +--------------+
        |                    |
        v                    v
   +-----------+        +------------+
   | Post-link |        | Runtime    |
   | rewriting |        | hooking /  |
   | (BOLT,    |        | injection  |
   |  e9patch, |        | (Frida,    |
   |  Dyninst) |        |  ptrace)   |
   +-----------+        +------------+

Legend:
  OFFLINE : analyzes or transforms bytes without executing the full program
  ONLINE  : observes/modifies behavior as instructions execute
  HW      : CPU provides periodic or filtered events; tool correlates to code
```

---

## 2. Static binary analysis tools

### 2.1 Disassemblers: what they do

A **disassembler** maps **bytes in executable sections** to **mnemonics** for a given ISA (x86, AArch64, RISC-V, …). The core difficulty is not decoding individual instructions (for fixed-length ISAs it is trivial; for x86 it is intricate but standardized) but **partitioning the byte stream into code vs data** and **recovering control flow** through **indirect** transfers.

Common outputs:

- **Linear disassembly**: start at entry point, decode forward; fast but wrong if execution skips into the middle of instructions or if data is interleaved (typical for hand-written assembly, some packers, jump tables).
- **Recursive disassembly**: follow control-flow edges from known code pointers; more accurate, still imperfect with computed jumps, self-modifying code, or opaque predicates.

### 2.2 Representative tools

| Tool | Role | Notable traits |
|------|------|----------------|
| **`objdump -d`** | CLI disassembler (Binutils / LLVM) | Ubiquitous, scriptable, no GUI; quality depends on symbol/debug info |
| **Ghidra** | Disasm + decompiler + SDK | Free, NSA-maintained; strong for batch RE and collaborative workflows |
| **IDA Pro** | Disasm + decompiler ecosystem | Industry standard for malware/RE; Hex-Rays decompiler is widely used |
| **Binary Ninja** | Disasm + IL + API | Clean IL, good automation APIs, collaborative features |
| **radare2 / rizin** | CLI-first RE framework | Extremely composable; steep learning curve; great for scripting |

These tools also provide **navigation** (xrefs, strings, imports), **type inference**, **signatures**, and **patching**—not just raw listing dumps.

### 2.3 Decompilers: Ghidra vs Hex-Rays

**Decompilers** lift machine code to a **higher-level intermediate representation** resembling **C-like** structure: variables, loops, `if/else`, function boundaries.

- **Ghidra decompiler**: bundled; quality is good for many compiler-generated binaries; extensible with scripts.
- **Hex-Rays (IDA)**: commercially licensed; long track record on difficult binaries; integrates tightly with IDA’s database.

Decompilation is **heuristic**: recovery of types, structs, and variable lifetimes may not match the original source even when behavior matches.

### 2.4 Control flow graph (CFG) reconstruction

A **basic block** is a **maximal sequence** of instructions with:

- a **single entry** (first instruction executed only via that point),
- **straight-line** execution until a **terminator** (branch, call that returns, halt, trap).

A **CFG** is a directed graph **BBs → edges** where an edge `(A → B)` means **execution may transfer** from the last instruction of `A` to the first of `B`.

**Challenges:**

- **Direct branches**: easy if targets are PC-relative and decoded correctly.
- **Indirect jumps** (`jmp *%rax`, virtual calls): need **pointer analysis**, jump tables from compilers, or hints from relocations/debug info.
- **Exception handling**: landing pads (LLVM `landingpad`, Itanium LSDA) create implicit edges.
- **Overlapping / ambiguous code** (rare in typical Linux server binaries): fundamental ambiguity.

### 2.5 ASCII diagram: CFG recovery from machine code

```
  Byte stream in .text (conceptual addresses)
  ======================================================
  0x1000:  push %rbp                    \
  0x1001:  mov %rsp,%rbp                |  Basic Block 0 (B0)
  0x1004:  cmp $0,%edi                  |  terminator: conditional jcc
  0x1007:  je  0x1020  -------------------+-----+
  0x1009:  mov $1,%eax                  |     |  false edge
  0x100e:  jmp 0x1025  -------------------+--+  |
  ...                                       |  |
  0x1020:  xor %eax,%eax  <----------------+  |  true edge (fall-through
  0x1022:  ...                     from je)     |   reordering may differ)
  0x1025:  ...  <-------------------------------+-- merge (B2)

  Resulting CFG (schematic):

        +-----+
        | B0  |
        +-----+
         /   \
        v     v
      +---+  +---+
      |B1 |  |B1'|   (compiler may layout many variants; edges recovered)
      +---+  +---+
        \     /
         v   v
        +--------+
        |   B2   |
        +--------+

  Tool pipeline inside a disassembler:

    Decode insns --> Identify leaders (jump targets) --> Split BBs
           |                                      |
           v                                      v
    Relocations / EH metadata ------------> Refine indirect edges
           |
           v
       Optional: merge bogus splits, lift to functions (call graph)
```

---

## 3. Dynamic binary instrumentation (DBI)

### 3.1 What DBI is

**DBI** interposes on **instruction execution** by:

1. **Taking control** when the CPU would run application code (OS + CPU features + runtime library).
2. Emitting **instrumented code** (a **code cache / translation cache**) that executes:
   - the original semantics **plus** callbacks (analysis routines),
   - or **modified semantics** (security, fuzzing mutators, dynamic optimization).

The application often **believes** it is running at certain addresses; the DBI may **virtualize** addresses or **redirect control** through trampolines. Transparency varies by tool (some DBIs strive for strong transparency; others accept visible differences).

### 3.2 Pin (Intel)

**Pin** is a **proprietary** (research and commercial-friendly) framework from Intel for **IA-32, x86-64, limited other targets** historically. You write **Pintools** in C/C++ with APIs to instrument at **instruction**, **basic block**, **trace**, or **routine** granularity.

Characteristics:

- Extremely capable for research (cache simulators, race detectors, call tracers).
- **Overhead** can be low for lightweight tools or very high for per-instruction analysis.

### 3.3 DynamoRIO

**DynamoRIO** is an **open-source** DBI runtime emphasizing **efficiency** and **extensibility**.

- Clients register **callbacks** for events (basic block, thread start/end, module load/unload).
- Uses **basic block traces**, **code caches**, **indirect branch lookup tables**.

Use cases: security analysis, record-replay helpers, custom coverage engines.

### 3.4 Valgrind and tool architecture

**Valgrind** is an **instrumentation framework** with a **middle-end** that lift machine code to an **intermediate representation (VEX)**, on which tools operate.

Conceptual layering:

```
  +--------------------------------------------------------+
  | Application + linked libraries (.text as normal ELF)  |
  +--------------------------------------------------------+
                            |
                            v
  +--------------------------------------------------------+
  | Valgrind core: scheduling, memory manager, syscall      |
  | interface, signal delivery, thread model                |
  +--------------------------------------------------------+
                            |
                            v
  +--------------------------------------------------------+
  | Front end: decode ISA -> VEX IR (guest state modeled)   |
  +--------------------------------------------------------+
                            |
              +-------------+-------------+
              |                           |
              v                             v
    +------------------+           +------------------+
    | Memcheck         |           | Callgrind        |
    | (shadow memory)  |           | (call graph +   |
    |                  |           |  cache sim)      |
    +------------------+           +------------------+
              |                           |
              v                             v
    +------------------+           +------------------+
    | Cachegrind       |           | Helgrind / DRD     |
    | (cache miss sim) |           | (thread errors)     |
    +------------------+           +------------------+

  Common theme: every guest load/store or branch runs through
  instrumented VEX-basic blocks in Valgrind’s JIT engine.
```

**Brief tool roles:**

- **Memcheck**: tracks **definedness** bits per byte; flags uninitialized reads, invalid frees, overlaps, many uses of freed memory. Heavyweight, still the default for “why is my C wrong?”.
- **Callgrind**: records **function-level costs** + optional cache simulation; pairs with **KCachegrind** visualization.
- **Cachegrind**: focuses on **L1/D1/LL** cache behavior modeling (instruction/data/last-level).
- **Helgrind** / **DRD**: thread errors (data races, lock order); different tradeoffs (Helgrind vs DRD annotations and performance).

### 3.5 Frida (runtime hooking)

**Frida** is not “just” a classical DBI like Pin/DynamoRIO, but overlaps heavily for **practitioners**:

- Embeds a **JavaScript** runtime in the target process (via injection).
- Can **replace** functions, **hook** on enter/leave, read/write memory, trace methods—popular on **Android**, **iOS**, **desktop**.

Mechanisms combine **code patching**, **trampolines**, sometimes **JIT**, depending on platform and stub strategy.

### 3.6 How DBI works under the hood (caching and JIT)

Typical execution loop:

1. **Dispatch / fragment lookup**: when control reaches not-yet-seen guest code, allocate a **translation fragment** (basic block / trace / superblock).
2. **Decode + optimize + instrument**: build IR, apply tool insertions, allocate machine code in the **code cache**.
3. **Link fragment**: patch direct branches **between cached fragments**; indirect branches consult **lookup tables**.
4. **Execute from code cache** until:
   - **exit stub** to dispatcher (system call, SIGSEGV handling, trace end),
   - **cache flush** (self-modifying code rare cases, explicit tool request).

**ASCII: DBI architecture**

```
  Guest binary (original addresses)        Instrumentation tool
  ============================             =====================
        |                                           |
        |  Control transfer enters DBI            |  Registers callbacks,
        |  from kernel (e.g. ptrace) or            |  analysis routines,
        |  injected bootstrap                      |  policies
        v                                           v
  +-------------------+                     +-------------------+
  | Dispatcher / VM   | <------------------ | Tool metadata      |
  | (find fragment)   |                   | (what to insert)   |
  +-------------------+                     +-------------------+
           |
           | miss in code cache
           v
  +-------------------+
  | JIT / translator  |
  | guest bytes ->    |
  | host instrumented |
  | machine code      |
  +-------------------+
           |
           | commit fragment
           v
  +-------------------+
  | Code cache (RWX   |
  |  or W^X staged)   |---- fast path ----> run instrumented path
  +-------------------+
           ^
           |
  +-------------------+
  | Indirect branch    |
  | resolution table   |
  +-------------------+

Cost knobs:
  - Granularity: per-instr vs per-BB vs per-trace
  - State virtualization: how faithfully flags/registers modeled
  - Memory metadata: shadow pages, bitmaps (Memcheck style)
```

---

## 4. Binary rewriting

### 4.1 What binary rewriting is

**Binary rewriting** transforms a **linked** program image by **changing instruction bytes**, **relocations**, **metadata**, or **layout**—producing a new binary with (hopefully) **preserved semantics** and **desired property** (faster, safer, smaller instrumented).

It differs from **source-level** or **IR-level** compilation changes: the input is already **low-level**, possibly **stripped**.

### 4.2 Static vs dynamic rewriting

| Style | When transformation applies | Pros | Cons |
|------|-----------------------------|------|------|
| **Static** (offline) | Before next run; new file on disk | Predictable overhead at runtime; can ship artifact | Must respect relocations, PLT, PIC, CFI, unwind info |
| **Dynamic** (loader / JIT) | At load time or on demand | Can specialize per environment | Complex integration; security policy on W^X pages |

**BOLT** is primarily **offline static rewriting** of `.text` (plus associated metadata updates), driven by **profiles**.

### 4.3 Challenges

1. **Indirect branches and calls**: unknown targets complicate reachability and safety if you move code.
2. **Data inlined in `.text`**: jump tables, literals, PC-relative tables; disassembler may confuse data as code.
3. **PC-relative addressing**: RIP-relative LEA and literals must be **rewritten** if code moves.
4. **Position independence (PIE/PIC)**: GOT/PLT interactions; linker assumptions.
5. **Exception handling / unwind**: CFI + `.eh_frame`, `.gcc_except_table` must stay **coherent** after moving instruction ranges.
6. **Debug information**: if addresses change, debug line info must be **adjusted** (or abandoned).
7. **Self-modifying code / JIT guests**: generally unsupported or requires special treatment.
8. **Checksums / signatures**: anti-tamper may reject rewritten binaries.

### 4.4 Tools (selected)

| Tool | Summary |
|------|---------|
| **BOLT** | Profile-guided **re-layout** and **micro-opts** of **fully linked** ELF binaries; LLVM-integrated. |
| **Dyninst** | Academic/industrial binary editing API; static + dynamic modes; patch instrumentation points. |
| **e9patch** | Static **binary patching** with emphasis on **non-destructive** rewriting and **scalability** for instrumentation. |
| **RetroWrite** | Automation for **rewriting** in security/research contexts (not a drop-in “speed up my server” tool like BOLT). |

---

## 5. BOLT in depth

### 5.1 What BOLT is

**BOLT** (**Binary Optimization and Layout Tool**) is a **post-link optimizer** that consumes **fully linked executables or shared objects** (notably **x86-64 ELF on Linux**) and rewrites them for better **CPU frontend** performance—primarily **instruction cache (I-cache)** and **ITLB** locality—guided by **profile data**.

BOLT lives under the **LLVM umbrella** (`bolt/` in the monorepo). The user-facing driver is typically **`llvm-bolt`**.

### 5.2 Why BOLT exists: PGO at binary level

Traditional **profile-guided optimization (PGO)** in **Clang/LLVM** happens **before** the final bytes are laid out:

- IR-level instrumentation or sampling produces profiles.
- The compiler uses them for **inlining**, **block layout**, **register allocation hints**, etc.

**Post-link layout** is still constrained by:

- **Linker script** decisions, **section ordering**, **hot/cold splitting** at object granularity, **alignment** choices, and historical **incremental linking** habits.
- **Cross-TU effects**: the hottest call edges may not correspond to object file boundaries the linker respects.

**BOLT operates where the “real” layout is fixed**—after all objects merge—so it can **reorder functions and basic blocks** with global visibility of **actual executed edges**.

### 5.3 BOLT vs compiler PGO (comparison)

| Aspect | Compiler PGO (Clang `-fprofile-*`) | BOLT |
|--------|-------------------------------------|------|
| **Input** | Source / LLVM IR / bitcode stage | Linked ELF binary |
| **Profile attach point** | Functions, LLVM BBs, calls | Executed **binary** addresses + (with LBR) **edges** |
| **Main wins** | Inlining, vectorization, specialization (semantic changes allowed at IR) | **Layout**: function order, BB order, **hot/cold splitting**, some **call-site** tweaks |
| **Rebuild from source** | Required for compile-time PGO | **Not required**; rewrites binary; ideal when rebuilding everything is expensive |
| **Preserves bit-exact semantics** | May change FP contract unless constrained | Strives for semantic preservation (still a linker-grade tool—bugs are bugs) |
| **Typical deployment** | Compile farm + LTO/PGO training | **Production binary** + **perf** workload capture |

They are **complementary**: Meta and others often use **ThinLTO + PGO + BOLT** chaining.

### 5.4 How BOLT achieves speedups (intuition)

Modern CPUs spend a surprising fraction of time in the **frontend**—**fetching** and **decoding** instructions—especially for large binaries (LLVM, Chrome-class, data center services).

If **hot code** is **scattered** across many cache lines and pages:

- **I-cache** lines hold cold instructions while hot code thrashes.
- **ITLB** misses rise when hot code spans many pages.
- **Taken branches** jump across cold regions.

BOLT **clusters hot code** contiguously, splits **cold** tails to **out-of-line** regions, **reorders functions** to keep callers/callees closer, and applies **micro-optimizations** (PLT, some branch forms) that reduce predictable overhead.

### 5.5 ASCII: BOLT’s architecture (conceptual modules)

```
                          +-----------------------------+
                          |  Fully linked ELF input      |
                          |  (executable or .so)         |
                          +-----------------------------+
                                      |
                                      v
                    +---------------------------------------+
                    |  Front-end: load ELF, disassemble     |
                    |  Build BB-level CFGs + call graph     |
                    +---------------------------------------+
                                      |
                    +-----------------+---------------------+
                    |                 |                     |
                    v                 v                     v
              +-----------+    +-------------+      +---------------+
              | Profile    |    | Pass mgr    |      | Diagnostics   |
              | ingestion  |    | (analysis & |      | (stats, viz,  |
              | (perf.fdata|    |  rewrite)   |      |  YAML logs)   |
              |  etc.)     |    |             |      +---------------+
              +-----------+    +-------------+
                    |                 |
                    +--------+--------+
                             v
                 +-------------------------+
                 | Rewriter / emitter      |
                 | - allocate new code     |
                 | - patch PC-rel/reloc    |
                 | - update EH / symbols   |
                 +-------------------------+
                             |
                             v
                 +-------------------------+
                 | Optimized ELF output    |
                 | + optional BAT map      |
                 +-------------------------+
```

---

## 6. How BOLT works

### 6.1 Step 1: Disassembly and CFG reconstruction

BOLT’s disassembly front-end:

- Parses **ELF segments/sections** containing executable code.
- Identifies **function symbols** if present (`STT_FUNC`), else uses **heuristics** / unwind info.
- Builds **CFGs** at the **basic block** granularity and links **call edges** (direct + some indirect patterns with profile help later).

Debug / unwind info improves boundary detection; stripped binaries are harder but not always impossible.

### 6.2 Step 2: Profile data ingestion (perf, LBR)

Primary Linux workflow:

1. Run the binary under **`perf record`** with **branch stacks** or **`LBR`** facilities where available.
2. Convert to BOLT’s textual **\*.fdata** with **`perf2bolt`** (LLVM tool).

The profile annotates **BB frequencies**, **edge frequencies**, and **call samples**, enabling **hotness sorting** and **layout algorithms**.

### 6.3 Step 3: Optimization passes (selected)

| Pass / feature | Purpose |
|----------------|---------|
| **Function reordering** + **hot/cold splitting** | Group hot functions; move cold fragments away |
| **Basic block reordering** | **Fall-through** hot paths; reduce taken branches along steady state |
| **Function splitting** | Isolate **cold** portions (error paths, unlikely traps) |
| **ICache clustering / “Hugify”** | Pack hot code to reduce conflict misses (implementation evolves with LLVM versions) |
| **PLT optimization** | Reduce indirect PLT overhead for hot external calls where safe |
| **Indirect call promotion** | Turn indirect calls into **direct** when target is statistically dominant (guarded) |
| **SCTC** | **Simplified conditional tail-call** transformation opportunities for layout-friendly tail paths |

Exact availability and flags depend on **LLVM/BOLT version**—always consult **`llvm-bolt -help`** for your build.

### 6.4 Step 4: Binary rewriting and emission

Emission merges:

- **New `.text` layout** (often **out-of-place** rewriting: allocate new executable sections or regions).
- Updated **PC-relative** fixups, **call relocations**, **jump tables** if moved.
- **Exception handling metadata** rewriting to match new code ranges.
- Optional **symbol table** / **debug line** adjustments (when enabled and supported).

### 6.5 ASCII: end-to-end BOLT pipeline

```
  (A) Training run                    (B) Profile conversion
  ================================    ============================

   perf record  + workload             perf.data
        |                                   |
        v                                   v
   perf.data -----------------------> perf2bolt ----> executable.fdata
                                                    \
                                                     `-> (maybe merge passes)


  (C) BOLT rewrite
  ================

   llvm-bolt  \
       -data=executable.fdata  \
       -o exe.bolt  \
       ...pass flags...
       --------> reads orig ELF + fdata
                      |
                      v
                 [ disasm + CFG ]
                      |
                      v
                 [ annotate hotness ]
                      |
                      v
           +----------+-----------+
           | layout optimizations |
           +----------+-----------+
                      |
                      v
                 [ emit ELF ]
                      |
                      v
                  exe.bolt + BAT (optional)
```

### 6.6 ASCII: hot/cold splitting (schematic)

```
Before (single function image, cold path interleaved):

  Address ---->
  +------------------------------------------------+
  | prologue + hot path + rarely-taken error path |
  | + epilogue (mixed in one contiguous region)     |
  +------------------------------------------------+
         many cache lines touched each invocation


After (conceptual; BOLT may outline cold blocks):

  HOT CLUSTER in rewritten .text
  +------------------+
  | prologue         |
  | hot fast path    |
  | tail goto cold   |---+
  +------------------+   |
         compact           |
                           v
                    +------------------+
                    | COLD OUTLINE     |
                    | error handling   |
                    | epilogue return  |
                    +------------------+

Benefits:
  - Hot portion occupies fewer cache lines
  - Fewer useless cold bytes compete for I-cache in steady state
```

---

## 7. Using BOLT practically

### 7.1 Building BOLT (from the LLVM monorepo)

BOLT is built as part of an **LLVM** build with the `bolt` project enabled. A typical **Ninja** CMAKE configuration (paths illustrative):

```bash
# Example only—adapt versions, install prefix, and host compiler.
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
mkdir build && cd build

cmake -G Ninja ../llvm \
  -DLLVM_ENABLE_PROJECTS="bolt;clang;lld" \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DCMAKE_INSTALL_PREFIX=/opt/llvm-bolt

ninja install
```

You should obtain **`llvm-bolt`**, **`perf2bolt`**, and related utilities in the install `bin/` directory.

> **Note:** macOS is not the primary BOLT target for Linux-style production workflows; tutorials and Meta’s public guidance focus on **Linux ELF x86-64**. Always verify support matrices for your LLVM version.

### 7.2 Collecting profile data with `perf`

A common pattern emphasizes **user-space cycles** + **branch stacks** so BOLT can recover **hot edges**:

```bash
perf record -e cycles:u -j any,u --call-graph=fp \
  -o perf.data -- ./your_program arg1 arg2
```

Flags vary by **kernel** and **CPU**:

- **`-j any,u`**: requests **branch stack** sampling (alias forms exist; kernels expose capabilities via `perf list` and `dmesg`).
- **`cycles:u`**: sample in **user** mode (reduce kernel noise for user-space binaries).

For short-running programs, increase sample density:

```bash
perf record -e cycles:u -j any,u -F 9999 -- ./your_program
```

If LBR depth is limited or unavailable, consult **`perf record` documentation** for your CPU—BOLT’s usefulness tracks **edge** information quality.

### 7.3 Converting perf data: `perf2bolt`

```bash
perf2bolt -p perf.data -o program.fdata ./your_program
```

`program.fdata` becomes the **`llvm-bolt -data=...`** input. Some workflows merge profiles from **multiple runs**.

### 7.4 Running BOLT: `llvm-bolt`

```bash
llvm-bolt ./your_program \
  -data=program.fdata \
  -o your_program.bolt \
  -reorder-blocks=ext-tsp \
  -reorder-functions=cd+ \
  -split-functions \
  -split-all-cold \
  -dyno-stats
```

**Interpreting flags:**

- **`-reorder-blocks=`** selects BB layout heuristic (`ext-tsp`, `cache`, `branch`, `normal`—names may vary slightly by version).
- **`-reorder-functions=`** selects **call graph** ordering (historically **`cd`** ≈ *call frequency*, **`hfsort+`** style variants appear in docs/releases).
- **`-split-functions` / `-split-all-cold`**: enable **hot/cold** outlining patterns.
- **`-dyno-stats`**: prints informational statistics (wording may evolve).

Always read **`-help`** on your exact binary; LLVM moves fast.

### 7.5 Common flags and options (non-exhaustive)

| Flag category | Examples | Role |
|--------------|----------|------|
| **I/O** | `-o`, `-data`, `--update-debug-sections` | Output path; profile; optional debug updates |
| **Scope** | `-bolt` subset selectors for linking multiple inputs | Whole-program vs per-DSO strategies |
| **Algorithm** | `-reorder-functions`, `-reorder-blocks` | Layout engines |
| **Micro-opts** | `-plt=`, `-icf=`, `-jump-tables=` | Target-specific safe transforms |
| **BAT map** | `-emit-bat` | Emit address translation for verification |
| **Relocations** | `-relocs` | Retain info needed for rewriting relocatable outputs |

### 7.6 Step-by-step workflow with a real program

**1) Build a representative binary**

- Prefer **`-O2`/`-O3`** and **LTO/PGO** if you already have those in production parity.
- Preserve **symbols** for easier debugging of BOLT issues (you can `strip` later).

**2) Capture `perf.data` on a realistic workload** (not “hello world” unless that *is* prod):

- Match input distributions; warm caches; avoid profiling **only** startup if steady-state matters.

**3) Convert with `perf2bolt`** using the **exact** ELF that produced the run (build-id discipline).

**4) Run `llvm-bolt`** with conservative flags first; confirm binary runs.

**5) Iterate** pass toggles (function splitting, ext-tsp) measuring each change.

**6) Ship `*.bolt`** as production artifact only after **CI tests + canaries**.

### 7.7 Measuring results

Good practice:

- Compare **wall time**, **CPU cycles**, **instructions retired**, **i-cache misses**, **ITLB misses** (`perf stat -e …`).
- Run **multiple iterations**, report **variance**; beware **turbo**, **ASLR**, **frequency scaling**.
- For services: compare **tail latency**—frontend stalls often hurt **p99** more than mean.

---

## 8. Profile collection

### 8.1 Hardware performance counters and LBR

**Performance monitoring units (PMUs)** expose counters (cycles, instructions, cache events). **Sampling** periodically records **instruction pointer (IP)** and optional **call stacks**.

**Last Branch Record (LBR)** is an **Intel** mechanism storing a small **ring buffer** of **recent branches** (from/to pairs). When sampled, you can reconstruct **which edge** was taken into a hot block—crucial for **edge profiling** without IR-level instrumentation.

AMD has different **branch sampling** facilities; workflows may differ—**verify CPU vendor documentation**.

### 8.2 `perf record` sampling modes (conceptual)

| Mode | What you get | When it helps BOLT |
|------|--------------|-------------------|
| **Cycle samples (user)** | Hot **addresses** | Basic block heat |
| **Branch stacks / LBR** | Hot **edges** | Superior **layout** for ext-tsp style algorithms |
| **Call graph (frame pointer)** | Caller/callee stacks | Augments **call graph** reasoning |

Pair modes with **frequency (`-F`)** and **duration** appropriate to workload length.

### 8.3 `perf2bolt` conversion caveats

- **Build IDs** / binary identity must line up—or you get mismatched profiles.
- Stripped vs unstripped: conversion cares about **load addresses** and **ELF structure**; mismatched PIE relocations confuse mapping.
- Very short runs may yield **sparse** profiles → **noisy layout**.

### 8.4 AutoFDO profiles

**AutoFDO** is Google’s **compiler-centric** pipeline leveraging **sampling** (often `perf`) to drive **GCC/Clang** optimizations without IR instrumentation.

Relationship to BOLT:

- **AutoFDO**: feeds **the compiler** while compiling (semantic + layout opportunities early).
- **BOLT**: consumes **runtime** address/edge profiles on **final** layout.

You may use **both** if your organization maintains compatible profile pipelines.

### 8.5 Instrumentation vs sampling (for profiling)

| Approach | Mechanism | Pros | Cons |
|----------|-----------|------|------|
| **Instrumentation PGO** | Compiler inserts counters | Exact counts, stable for thin loops | Rebuild + runtime hit during training |
| **Sampling (`perf`)** | PMU interrupts + IP/LBR | Low overhead, usable in prod (carefully) | Statistical noise; needs enough samples |
| **JIT/DBI counters** | DBI tool tallies blocks | Flexible | Much higher overhead; engineering complexity |

BOLT historically centers on **Linux `perf` + LBR-class edge info**, not compiler IR counters.

### 8.6 ASCII: LBR capture on a branchy execution trace

```
  Time -->
  Executed branches (conceptual):  A->B, B->C, C->D, D->C, C->E, ...

  CPU LBR ring buffer (depth N, small):
      +------+------+------+------+
  idx | B->C | C->D | D->C | C->E |  (newest at right)
      +------+------+------+------+

  On PMU overflow sampling event:
     sample IP + optional LBR stack snapshot
                     |
                     v
          perf records "was executing near hot site,
                         likely arrived via edge (D->C)"

  Aggregate many samples -> edge frequency estimates
                 |
                 v
             BOLT .fdata edge weights
```

---

## 9. BOLT optimizations explained

### 9.1 Function reordering: HFSort and C3 (intuition)

**HFSort** family algorithms (from **profile-driven call graph clustering** research) prioritize placing **heavy interaction** functions **adjacent** so **call distances shrink** and **code locality** improves.

**C3** (from Meta’s public discussions / code) refines clustering with **cache-aware** neighbor triples—approximating benefits of **grouping three mutually interacting** functions to mitigate **conflict misses** beyond pairwise greedy placements.

Exact implementations evolve; treat names as **anchors** to read **source comments / release notes** in your LLVM checkout.

### 9.2 Basic block layout: Extended TSP

**BB layout** can be modeled as a graph problem: nodes are blocks, weighted edges represent **transition frequencies**. **TSP-like** heuristics seek an ordering maximizing **fall-through** along hot edges (minimizing taken branches and improving fetch/decode streams).

**Extended TSP** formulations add terms for **cache line** transitions, **alignments**, and **fetch bundle** effects—moving beyond purely **branch-centric** metrics.

### 9.3 Cache-line optimization

Even perfect **branch** layout fails if hot blocks **map to the same cache sets** conflictingly. Optimizers may incorporate **linear address** modulo effects (set conflicts) into scoring, attempting to spread **conflicting** hot regions.

### 9.4 Code alignment

**NOP** insertion and **alignment** to **bundle** or **cache line** boundaries can help **uop fetch** and **decode** on some microarchitectures—but can also **inflate code size**. BOLT supports policies balancing **size vs alignment** gains.

### 9.5 Call continuation optimization

When a **call** returns, execution resumes at the **continuation block**. Placing the **likely return block** near **callee hot code** or avoiding **anticache line splits** at calls can shave frontend overhead. BOLT includes **call-centric** layout considerations in its scoring.

### 9.6 Stale profile handling

Profiles age when code changes. Mitigations:

- **Fresh training** on each major release.
- **Hashing / build IDs** to refuse mismatched profiles.
- **Robust layout**: don’t overfit low-confidence edges; some passes degrade gracefully if edges are missing.

---

## 10. BOLT in production

### 10.1 Meta’s use case

Meta publicly reported **significant CPU savings** applying BOLT to large binaries (compilers, datacenter services) on top of **already strong** compile-time optimizations. The recurring theme: **frontend bound** workloads on **huge** `.text` footprints benefit enormously from **global layout**.

Use internal engineering talks and LLVM dev meetings for the freshest numbers—**do not** treat blog posts as benchmarks for your SKU.

### 10.2 Clang/LLVM bootstrap with BOLT

The LLVM community documented workflows optimizing **Clang** with **PGO + ThinLTO + BOLT**, showing meaningful speedups to compiler throughput. This matters because **Clang itself** is both:

- a workload sensitive to I-cache, and
- a toolchain consumers rebuild often.

### 10.3 Linux kernel with BOLT

Experimentation exists to **post-link optimize kernel images** / selected text sections. This is **operationally delicate** (panic safety, unwinding, special sections, objtool integration). Treat public kernel ML threads and LLVM docs as **primary** sources; expect **moving constraints** across kernel versions.

### 10.4 Performance numbers and case studies

Rules for responsible reading:

- **Microarchitecture matters** more than ISA family: **Rome vs Genoa vs Sapphire Rapids** changes i-cache behavior.
- **Binary size & linkage** dominates: **7 MB** vs **700 MB** `.text` binaries see different gains.
- Measure **your** workload; quoted **5–20%** wins are **not universal guarantees**.

---

## 11. Other binary optimization tools

### 11.1 Propeller (Google)

**Propeller** is Google’s **post-link** optimization approach emphasizing **relinking** with **basic block sections** (`-ffunction-sections` style at **section per BB** granularity) and **linker** awareness to **shuffle** layout using **profiles**.

Think: **co-design** the compiler/linker to surface fine-grained relocatable fragments.

### 11.2 AutoFDO

Sampling-driven **compiler** profile (`perf` → profile → re-compile). Excellent for **semantic** decisions but cannot retroactively fix **purely linker-final** layout problems the compiler never saw cohesively.

### 11.3 BOLT vs Propeller (high level)

| Topic | BOLT | Propeller |
|------|------|-----------|
| **Primary locus** | Binary rewriting after link | Compiler emits BB sections; linker relocates using profile |
| **Toolchain coupling** | Works on many existing ELFs (constraints apply) | Needs **propeller-aware** compile+LTO+link pipeline |
| **Flexibility vs integration** | Strong when you cannot revamp whole build | Strong when you can restructure build & linker graph |

### 11.4 When to use what

- **You control Clang/LLVM+LTO end-to-end** and can adopt Propeller-style builds: consider **Propeller** integrated plans.
- **You need to optimize **already linked** artifacts**, third-party heavy binaries, or lack Propeller build churn: **BOLT**.
- **You still need semantic PGO**: **compile-time PGO/AutoFDO** remains foundational.

---

## 12. Practical examples

### 12.1 Complete BOLT workflow (template)

```bash
# 0) Versions
llvm-bolt --version
perf --version

# 1) Build program (example)
clang++ -O3 -g -o mybin mysrc.cc

# 2) Profile
perf record -e cycles:u -j any,u -F 4000 \
  -o perf.data -- ./mybin --your realistic args

# 3) Convert
perf2bolt -p perf.data -o mybin.fdata ./mybin

# 4) Optimize
llvm-bolt ./mybin \
  -data=mybin.fdata \
  -o mybin.bolt \
  -reorder-blocks=ext-tsp \
  -reorder-functions=cd+ \
  -split-functions \
  -split-all-cold \
  -dyno-stats

# 5) Measure
perf stat -r 5 -e cycles,instructions,cache-misses,iTLB-load-misses \
  ./mybin --args
perf stat -r 5 -e cycles,instructions,cache-misses,iTLB-load-misses \
  ./mybin.bolt --args
```

### 12.2 Analyzing BOLT’s output: BAT (BOLT Address Translation)

When **`-emit-bat`** (or equivalent in your version) is enabled, BOLT can emit metadata mapping **original addresses** to **rewritten** addresses—useful for:

- correlating **`perf`** profiles from old vs new binaries,
- debugging crashes symbolicated against **pre-BOLT** symbols,
- tooling that needs **stable conceptual addresses**.

Treat BAT as a **versioned sidecar** to your deployment artifact.

### 12.3 Verifying correctness

- **Differential testing**: run **golden** integration suites on both binaries.
- **Checksum intermediate results** for deterministic workloads.
- **Stress rare code paths**; layout changes can **expose** latent UB—legally “your bug,” practically “you’ll get paged.”
- For floating point, watch **vector register spills** changing results if you had **strictness bugs**—BOLT aims not to change semantics, but **don’t skip tests**.

### 12.4 Benchmarking methodology

1. **Pin threads / isolate cores** for microbench rigor where possible.
2. Warmup runs **outside** timed region.
3. Report **mean + stdev**; use enough iterations.
4. Track **frequency** (`turbostat`) and disable **ASLR** only if your policy allows (microbench only).
5. For **services**, measure **p95/p99** latency and **error budgets**, not just RPS.

---

## Closing orientation map (ASCII)

```
 Goals \ Tool classes
 ------------------------------------------------------------------
 Understand code     | disasm/decompiler (static)
 Find runtime truth  | DBI + sanitizer + tracers (dynamic)
 Optimize layout     | BOLT / Propeller / linker tricks (post-link)
 Optimize semantics  | Compiler PGO / AutoFDO (compile-time)
 Observe cheaply     | perf PMU sampling + LBR (hardware-assisted)
```

BOLT is the **specialist** for **instruction-locality** on **massive** binaries once **everything else** (algorithms, data structures) is already excellent—**it squeezes the machine** by **repackaging** the bytes you already proved are correct.

---

## Further reading (external, verify against your LLVM version)

- LLVM documentation: [https://llvm.org/docs/](https://llvm.org/docs/) and search for **BOLT**.
- `llvm-project/bolt` **README** and **test/** tree for `-reorder-*` regression examples.
- Linux `perf` wiki / `man perf-record` for **LBR** capabilities on your kernel.
- Academic papers on **profile-guided basic block ordering**, **cache-aware code placement**, and **call graph clustering** for intuition behind HFSort / TSP formulations.

---

*Tutorial compiled for educational purposes; specific flags and supported targets change across LLVM releases—always prefer the documentation bundled with your toolchain build.*
