# Processor Instruction Set Architectures (ISA): A Deep Tutorial

This document is a detailed, educational survey of **Instruction Set Architectures**—the contract between software and hardware. It compares major families, explains encoding and addressing, privilege and memory models, evolution of extensions, and how to read vendor manuals. Examples mix **x86-64**, **ARM AArch64**, **RISC-V**, and **classic MIPS** where they illustrate differences clearly.

---

## 1. What Is an ISA?

### 1.1 Definition and Role as Hardware–Software Interface

An **Instruction Set Architecture (ISA)** is the **programmer-visible** specification of a processor family: the set of **instructions**, their **encodings**, **operand types**, **registers**, **memory and I/O model**, **privilege and exception model**, and often **optional extensions** (SIMD, crypto, virtualization).

The ISA is *not* the same as **microarchitecture** (pipelines, caches, branch predictors, out-of-order execution). Many different silicon implementations can share one ISA; conversely, one chip vendor may ship several ISAs in different product lines.

**Hardware–software contract** means:
- **Software** (compilers, OS, hand-written assembly) assumes certain instructions behave as documented.
- **Hardware** must make those behaviors observable (within defined timing and power constraints), or the implementation is non-conforming.
- **Binary compatibility** across CPU generations is valuable: old binaries keep running because the ISA is stable (though extensions add new instructions over time).

### 1.2 Where the ISA Sits in the System Stack

```
+=====================================================================+
|                    APPLICATIONS & RUNTIMES                          |
|  (browsers, databases, games, JVM, Python, ...)                     |
+==============================+======================================+
                               |
+==============================v======================================+
|                 HIGH-LEVEL LANGUAGES & COMPILERS                    |
|  C/C++/Rust/Go/Swift → LLVM/GCC → target ISA                        |
+==============================+======================================+
                               |
+==============================v======================================+
|              OPERATING SYSTEM KERNEL & FIRMWARE                     |
|  Uses privileged ISA: page tables, traps, device MMIO, ...          |
+==============================+======================================+
                               |
+==============================v======================================+
|  +-------------------------------------------------------------+    |
|  |              ABI / CALLING CONVENTION (NOT ISA)             |    |
|  |  Which registers are arguments/callee-saved, stack layout   |    |
|  |  OS + compiler agree; per-OS on same ISA can differ         |    |
|  +------------------------------+------------------------------+    |
|                                 |                                   |
|  +==============================v===============================+   |
|  |         INSTRUCTION SET ARCHITECTURE (ISA)  <-------- YOU    |   |
|  |  • Opcodes & encodings   • User vs supervisor instructions   |   |
|  |  • Register file         • Virtual memory model              |   |
|  |  • Addressing modes      • Exceptions & interrupts           |   |
|  |  • Memory consistency    • Optional extensions (SIMD, ...)   |   |
|  +==============================+===============================+   |
|                                 |                                   |
+==============================v======================================+
|           MICROARCHITECTURE (IMPLEMENTATION DETAIL)                 |
|  Fetch/decode, rename, OoO scheduler, caches TLB, µops, ...         |
+==============================+======================================+
                               |
+==============================v======================================+
|              PHYSICAL SILICON (RTL, layout, process node)           |
+=====================================================================+
```

**Takeaway**: The ISA is the **stable boundary** where software can reason about correctness. Everything below can change every product generation as long as it preserves ISA-visible behavior (modulo timing side channels, which are a security research topic of their own).

### 1.3 Why ISAs Matter

| Concern | Why ISA choice matters |
|--------|-------------------------|
| **Software porting** | Recompilation or translation cost when moving between ISAs (ARM server vs x86 desktop, embedded RISC-V). |
| **Binary ecosystems** | x86-64 has decades of proprietary and open binaries; mobile is dominated by AArch64. |
| **Energy & area** | Simpler decode (e.g., fixed 32-bit RISC) vs dense variable-length CISC affects front-end power. |
| **Real-time & safety** | Predictable timing, MMU presence, PMP/MPIE models differ by ISA + profile. |
| **Licensing & openness** | ARM: architectural license + cores; x86: two main vendors; RISC-V: open standard, many implementers. |
| **Long-term evolution** | Whether vector, atomics, virtualization arrive as **mandatory**, **optional**, or **profile-defined** pieces. |

---

## 2. ISA Design Philosophy

### 2.1 CISC vs RISC (Conceptual)

**CISC (Complex Instruction Set Computer)** historically emphasized **dense encoding** and **rich memory-operand instructions** (e.g., `add [mem], reg`). Microcoded machines amortized complexity. Modern “CISC” CPUs often decode x86 into internal **RISC-like micro-operations (µops)**.

**RISC (Reduced Instruction Set Computer)** emphasized **load/store architectures** (only explicit `load`/`store` touch memory), **fixed or regular instruction length**, **large register files**, and **pipelining friendliness**. Compilers moved complexity from hardware to software.

```
                        CISC heritage                    RISC heritage
                        -----------                    -------------

   Typical memory                  Typical memory
   operands in ALU                 operands: register-only
          |                                |
          v                                v
   +-------------+                  +-------------+
   | reg  <-->   |                  |   LOAD      |
   |   memory    |                  |   ALU       |
   +-------------+                  |   STORE     |
                                    +-------------+
   Variable-length                    Fixed 32-bit
   instr. common                      instr. common (classic)


ASCII "spectrum" (simplified, not a strict taxonomy today):

  DENSE / FEW INSTRs                  REGULAR / MANY SIMPLE INSTRs
  complex addressing                   load/store + simple ALU
        |                                         |
   x86-64 --------------------------------- RISC-V, AArch64, MIPS32
        \__________ modern decode to µops ______/
```

**Modern reality**: Microarchitecture blurs the pure dichotomy. **AArch64** is elegant and RISC-like yet has sophisticated addressing; **x86-64** is CISC at the ISA level but often RISC-like inside the core.

### 2.2 VLIW and EPIC

**VLIW (Very Long Instruction Word)** packs **multiple operations** into one wide instruction word. The compiler (or specializer) schedules **which operations issue together**; hardware uses simpler issue logic than dynamic superscalar scheduling.

```
  Traditional superscalar (dynamic):          VLIW bundle (static):

  fetch -> decode -> pick 0-4 ops           fetch bundle
           (rename)    dynamically              |
                    |                           v
                    v                   +------+------+------+------+
              execute out-of-order      | op A | op B | op C | op D |
                                        +------+------+------+------+
                                        (all issue same cycle if legal)
```

**Trade-offs**:
- **Pros**: Potentially lower control complexity; good for embedded DSPs and some GPUs.
- **Cons**: **Code density** and **binary compatibility** suffer when the machine width changes; compiler complexity; **NOP padding** when slots cannot be filled.

**EPIC (Explicitly Parallel Instruction Computing)** (used in **Intel Itanium / IA-64**) extended ideas with **predication**, **explicit parallelism hints**, and large bundles. It aimed to push scheduling to the compiler; market uptake was limited versus x86-64 and ARM.

### 2.3 Trade-offs Table

| Dimension | Tends toward CISC / legacy | Tends toward RISC / regular |
|-----------|---------------------------|-----------------------------|
| **Instruction length** | Variable (x86) | Fixed 32-bit (many RISCs); 16-bit compression optional (RVC, Thumb) |
| **Decode complexity** | Higher front-end | Simpler decode |
| **Code size** | Often smaller for some idioms | Larger without compression extensions |
| **Memory ops in ALU** | Common (x86) | Unusual (load/store ISA) |
| **Compiler role** | Easier for some legacy patterns | Must schedule loads/stores explicitly |
| **Extensibility** | Opcode space pressure | Reserved opcodes / standard extension slots |
| **Verification cost** | Complex corner cases | Often cleaner formal models (not always) |

---

## 3. Major ISAs in Detail

### 3.1 x86 / x86-64

**Lineage (selected milestones)**:

```
  8086 (1978) 16-bit  ->  80286 protected mode  ->  80386 32-bit
       -> Pentium superscalar -> Pentium Pro P6 µop internal core
       -> x86-64 (AMD64, ~2000; Intel EM64T) 64-bit GPRs, more regs
       -> SIMD & vector extensions (see §8)
       -> Security & virt extensions (VT-x, SGX phases, CET, ...)
```

**User-visible characteristics**:
- **Variable-length** instructions (1–15 bytes in 64-bit mode; complex prefixes).
- **Few architectural GPRs** in 32-bit era (EAX–EDX, ESI, EDI, EBP, ESP); **x86-64** adds **r8–r15** and uniform access to **rax–rdi** extended to 64-bit.
- **Flags register (RFLAGS)** with condition codes used by many integer ops.
- **Floating-point**: historical **x87 stack**; modern code uses **SSE/AVX** XMM/YMM/ZMM registers.
- **Segmentation** largely relegated to compatibility; **flat 64-bit** virtual address space is typical in modern OSes.

**Extension families (high level)**:
- **MMX**: SIMD packed integer mapped on **FP/MMX mm** registers (legacy path).
- **SSE** through **SSE4**: XMM (128-bit), scalar/packed FP and integer.
- **AVX / AVX2**: YMM (256-bit), three-operand non-destructive encodings many cases.
- **AVX-512**: ZMM (512-bit), **mask registers k0–k7**, **opmask predication**, new rounding/exception granularity; subsets (`F`, `CD`, `BW`, `DQ`, `VL`, …) vary by CPU model.

**Why it persists**: enormous software corpus, performance engineering investment, PC/server supply chain. Complexity is managed by superb microarchitectures and extensive tooling.

### 3.2 ARM

**ARM7 / classic ARM** (ARMv4–v6 era teaching examples):
- **32-bit ARM and 16-bit Thumb** (original): density vs performance trade-off.
- **Conditional execution**: many ARM instructions could be predicated with **4-bit condition field** (`EQ`, `NE`, `LT`, …), reducing branches in hand-tuned code.

**ARMv7-A** (32-bit application profile):
- **Cortex-A** cores (many models): superscalar, NEON SIMD, optional security extensions.
- **Thumb-2**: mixed 16/32-bit for better density.

**ARMv8-A / AArch64**:
- **64-bit** clean break: **31 GPRs** (X0–X30, **XZR/WZR** zero register), **PC not general-purpose** in ARMv8 (unlike some earlier ARM discussion patterns in textbooks).
- **NEON** mapped to **128-bit vector** registers **V0–V31** (as SIMD; also used for FP scalars).
- **SVE / SVE2** (optional): scalable vectors for HPC.

**Thumb / conditional model note**:
- **AArch64** **does not** carry the classic “most instructions are conditional” model; predicates are far more limited. Condition codes live in **NZCV**; conditional branches and selected flag-setting ops replace old patterns.

```
  ARM ISA family simplification:

  ARMv7-A 32-bit          AArch64 64-bit
  -------------           --------------
  ARM + Thumb           A64 instruction set
  heavy predication     focused on branches + flag ops
  NEON (optional)       NEON/SIMD mandatory in most app CPUs
```

### 3.3 RISC-V

**Open standard** (RISC-V International): royalty-free **ISA specification**, not a single vendor’s chip.

**Modular design**:
- **Base integer RV32I / RV64I** (or **E** embedded reduced): mandatory core.
- **Standard extensions** (letters): **M** integer multiply/divide, **A** atomics, **F** single float, **D** double float (**F+D** often paired), **C** compressed 16-bit instructions, **Zicsr** CSR access, **Zifencei** instruction-fetch fence, **B** bit-manip, **Vector** extension **V**, **Hypervisor** **H**, **Privileged** architecture separate spec, etc.

**Profiles**: e.g., **RVA20**, **RVA22** application profiles define **mandatory** base + extensions for interoperable software stacks (POSIX-class OS, etc.).

**Philosophy**:
- **Separates user and privileged ISA** into different documents.
- **No implicit status flags** for generic integer ops: compares set destination registers (typical), which simplifies dependency tracking in simple cores.

### 3.4 MIPS (Classic Teaching ISA)

**Classic MIPS32/64** (educational and historical embedded):
- **32 GPRs** (some reserved by convention: `$zero`, `$at`, v0–v1, a0–a3, etc.).
- **Load/store**, **3-register** ALU ops, **HI/LO** for multiply (`mult`) legacy path.
- **Branch delay slot**: instruction **after** a taken branch still executes (unless explicitly disabled on some implementations). **Load delay** appeared in early MIPS; modern cores often interlock but ISA retains delay-slot heritage in *branch*.
- **Fixed 32-bit** encoding, regular fields (R/I/J types).

```
  MIPS classic pipeline picture (5 stage) with branch delay:

  IF ID EX MEM WB
        |      \
        branch resolved here (classic)
        v
      slot instruction in EX (still executes if branch taken)


  Note: Delay slots complicate compilers and exceptions; many newer ISAs avoid them.
```

### 3.5 Brief Mention: PowerPC, SPARC, Itanium

| ISA | Notes |
|-----|--------|
| **PowerPC / Power ISA** | IBM POWER line, embedded **PPC** historically; strong server presence; AltiVec VSX SIMD. |
| **SPARC** | Sun/Oracle lineage; register windows; server/workstation history; open SPARC specs exist. |
| **Itanium (IA-64)** | EPIC VLIW-style bundles; Intel/Hewlett-Packard era; largely superseded in general market by x86-64. |

---

## 4. Instruction Encoding

### 4.1 Fixed-Length vs Variable-Length

```
  FIXED 32-bit (typical RISC):

  |<-------- 32 bits --------->|
  +-----------------------------+
  |  opcode / funct / immediates |
  +-----------------------------+

  Every fetch boundary aligns; decode is indexing.


  VARIABLE-LENGTH (x86):

  byte stream:  [prefix?][REX?][opcode][ModR/M][SIB][disp][imm]...

  Decoder walks bytes; instructions have **minimum** and **maximum** length.
  Density higher; decode wider and more complex.
```

### 4.2 Operand Fields: Opcodes, Registers, Immediates

Common pieces across ISAs:
- **Opcode** identifies operation (add, load, branch).
- **Source / destination register** indices (5–6 bits per reg on 32-reg files).
- **Immediate** (sign-extended or zero-extended constant embedded in instruction).
- **Function codes** (`funct3`/`funct7` in RISC-V) differentiate variants sharing an opcode.
- **Addressing mode bits** select base + index + scale + displacement (x86 ModR/M & SIB).

### 4.3 RISC-V 32-Bit Base Formats (RV32I / RV64I)

```
  R-type (register-register):

   31      25 24      20 19      15 14  12 11      7 6       0
  +------------+----------+----------+--------+----------+----------+
  |  funct7    |   rs2    |   rs1    | funct3 |    rd    |  opcode  |
  +------------+----------+----------+--------+----------+----------+

  I-type (immediate ALU, loads, JALR):

   31                   20 19      15 14  12 11      7 6       0
  +-----------------------+----------+--------+----------+----------+
  |       imm[11:0]       |   rs1    | funct3 |    rd    |  opcode  |
  +-----------------------+----------+--------+----------+----------+

  S-type (stores):

   31      25 24      20 19      15 14  12 11      7 6       0
  +------------+----------+----------+--------+----------+----------+
  | imm[11:5]  |   rs2    |   rs1    | funct3 | imm[4:0] |  opcode  |
  +------------+----------+----------+--------+----------+----------+

  B-type (branches) – immediate bits scrambled across fields by spec

  U-type (LUI/AUIPC) – high immediate + rd + opcode

  J-type (JAL) – jump immediate scrambled + rd + opcode
```

**Example (conceptual)**: `addi x5, x6, 3`
- **opcode** = OP-IMM; **funct3** = ADD; **rs1** = x6; **rd** = x5; **imm** = 3.

### 4.4 AArch64 Instruction Layout (Representative)

Many data-processing instructions use **31-bit fields** in a fixed 32-bit word:

```
  Typical DP (Register) – simplified; exact packing varies by encoding group:

   31  30 29 28        24 23 22 21 20 16 15 14 10 9  5 4    0
  +----+--------------+--+--+----+-----+------+-----+------+
  | sf |  opcode grp  |M |S | Rm  | imm | Rn   | Rd  | opc  |
  +----+--------------+--+--+----+-----+------+-----+------+

  sf = 0 -> 32-bit; sf = 1 -> 64-bit operation

  Loads/stores use separate families (scaled/unscaled, pair, literals).
```

**Example**: `ADD X0, X1, X2` packs opcode group identifying ADD, size 64, sources X1/X2, dest X0.

### 4.5 x86-64 Instruction Layout (ModR/M World)

Many ALU/legacy instructions use:

```
  [Prefixes] [REX] opcode ModR/M [SIB] [displacement] [immediate]


  REX prefix (64-bit extension) – when present:

    7    6 5 4   3   2   1   0
  +----+-----+---+---+---+---+
  |0100| W   | R | X | B |   |   (common pattern; details in Intel SDM)
  +----+-----+---+---+---+---+

  ModR/M byte:

    7 6   5 4 3     2 1 0
  +-------+---------+-----+
  | mod   | reg/op  | r/m |
  +-------+---------+-----+

  mod=11 often means register r/m
  mod=00/01/10 encode memory modes with disp0/disp8/disp32 flavors
  r/m=100 with mod!=11 often implies SIB follows

  SIB byte (scale/index/base):

    7 6   5 4 3   2 1 0
  +-------+-------+-----+
  | scale | index | base|
  +-------+-------+-----+
```

**Why it looks messy**: Decades of backward compatibility; prefixes for segment override (legacy), `REX` for 64-bit, `VEX`/`EVEX` for SIMD expansion, etc.

### 4.6 How the CPU Decodes Instructions (High Level)

```
  FETCH:  Read instruction bytes from L1I (or uop cache on x86).

  LENGTH / BOUNDARY:  x86 predecoder finds instruction boundaries;
                      RISC-V/ARM64 fetch 4 bytes.

  DECODE:  Split fields -> opcode map -> instruction ID
           expand to internal uops if needed

  REGISTER RENAME:  Map arch reg -> physical reg (OoO cores)

  SCHEDULE / EXECUTE:  Issue to ALU/AGU/FPU/vector units

  COMMIT:  Architected state updated in-order; exceptions precise at boundaries
```

**Key point**: **Decode width** and **branch prediction** dominate front-end performance on wide cores; variable-length ISA pays continuous decode tax, partially mitigated by **µop caches**.

---

## 5. Addressing Modes

### 5.1 Comprehensive List (Generic Taxonomy)

| Mode | Pattern | Typical use |
|------|---------|-------------|
| **Register** | `op dst, src1, src2` | ALU on registers |
| **Immediate** | `op dst, src, #imm` | Constants |
| **Direct / absolute** | `op [addr]` | Static globals (when supported) |
| **Register indirect** | `op [base]` | Pointer chasing |
| **Displacement** | `op [base + disp]` | Struct fields, locals |
| **Indexed** | `op [base + index]` | Arrays |
| **Scaled index** | `op [base + index*scale + disp]` | Arrays of structs |
| **PC-relative** | `op [PC + offset]` | Position-independent code, literals |
| **Pre/Post indexed** | `[base, #imm]!`, `[base], #imm` | Stack / loops (ARM family features) |

### 5.2 How Different ISAs Expose Modes

**x86-64**: Extremely flexible; `ModR/M + SIB + disp` encodes many combinations; RIP-relative addressing common for PIC data (`[rip+disp32]`).

**AArch64**: Base + immediate offset (with scaling rules), literal loads PC-rel, pre/post-index forms for loads/stores; `LDUR`/`STUR` for unscaled offsets in some cases.

**RISC-V**: Base+offset **only** for loads/stores; complex addressing compiled into **add** + load/store.

**MIPS**: Mostly `offset(base)` with 16-bit signed offset; advanced addressing via extra ALU instructions.

---

## 6. Privilege Levels and Execution Modes

### 6.1 x86 Rings (Conceptual)

```
                         PRIVILEGE LEVEL (CPL / RPL concepts)

   Ring 0  (most privileged)   -------- OS kernel
      |
   Ring 1                      \
   Ring 2                       | rarely used in mainstream OSes
      |                         /
   Ring 3  (user mode)         -------- applications

  Modern OS: mostly **kernel (0)** vs **user (3)**; hypervisors add roots.


ASCII stack of who can touch what (simplified):

  +----------------------------------+
  |  Hypervisor / System management  |  (VMX root, SMM, firmware)
  +----------------+-----------------+
                   |
  +----------------v-----------------+
  |         Ring 0 OS                |  page tables, MSRs, traps
  +----------------+-----------------+
                   |
  +----------------v-----------------+
  |         Ring 3 apps              |  syscall gate into ring 0
  +----------------------------------+
```

**Transitions**:
- **SYSCALL/SYSENTER**-style fast path into kernel dispatcher.
- **INT n**, **exceptions** (#PF page fault, #GP general protection), **interrupts** from APIC—vector through IDT (legacy picture) / modern interrupt subsystems.

### 6.2 ARM Exception Levels (AArch64)

```
  EL3 (Secure monitor) --- optional TrustZone monitor firmware
   |
  EL2 (Hypervisor) --- virtualization, stage-2 translation
   |
  EL1 (OS kernel)
   |
  EL0 (Application)

Typical phone: EL0 apps, EL1 RichOS, EL3 secure world firmware pieces.
Server with KVM: guests at EL1/EL0, host hypervisor EL2.
```

**VHE (Virtualization Host Extensions)**: lets host kernel run at **EL2** in some configurations to reduce world switches—see vendor kernel docs for exact usage patterns.

### 6.3 RISC-V Privilege Modes

Specification defines **Machine (M)**, optional **Supervisor (S)**, optional **User (U)**; **Hypervisor (H)** extension adds HS-mode and virtual VS/VU ideas for type-2 virtualization.

```
  Highest privilege ---- M-mode (firmware, platform control)
           |
           +---- S-mode (OS) --- if implemented
           |
           +---- U-mode (apps)

Control registers: **CSR** space; **medeleg/mideleg** can delegate traps.
```

### 6.4 Mode Transitions (Unified View)

| Purpose | x86-64 (Linux) | AArch64 (Linux) | RISC-V (Linux) |
|---------|----------------|-----------------|----------------|
| User→kernel syscall | `syscall` (or `int 0x80` legacy) | `svc #imm` (immediate is OS convention) | `ecall` |
| Kernel return | `sysret` / `iretq` paths | `eret` (exception return) | `sret` / `mret` (per privilege) |

```
  User execution
       |
       | syscall / svc / ecall
       v
  Kernel entry trampoline:
     - save user context (GPRs, PC, SP, ...)
     - switch to kernel page tables if needed
     - dispatch syscall table
       |
       v
  Return path (SYSRET/ERET/SRET depending on ISA):
     - restore user context
     - restore PC to next user insn
```

**Exceptions vs interrupts**: **Exceptions** are synchronous to instruction stream (faulting load, divide error). **Interrupts** are asynchronous device/timer events. Both raise privilege to a handler **defined by ISA + platform + OS**.

---

## 7. Memory Model

### 7.1 Memory Ordering: Strong vs Weak

**Strong (sequential consistency feel at source level—rarely literal SC hardware for all ops)**: x86-style **Total Store Order (TSO)** class behavior for normal writes/reads—simpler for programmers, more constraints on hardware.

**Weak**: **AArch64** and **RISC-V** have a **relaxed default memory model** for ordinary loads/stores; **ordering requires explicit barriers or acquire/release annotations**.

```
  Programmer's view:

  STRONGER hardware ordering         WEAKER hardware ordering
  (still need careful concurrency!) (more annotations)

  fewer fences for some idioms      explicit acquire/release/ fences
```

### 7.2 Barriers and Fences

Examples of concepts (exact mnemonics vary):
- **Full fence**: orders all prior memory ops before subsequent for some definition of “all”.
- **Acquire**: subsequent loads/stores cannot move before acquire load.
- **Release**: prior loads/stores cannot move after release store.
- **Store-store / load-load fences**: partial orders.

**RISC-V**: `fence` instruction with predecessor/successor sets (`rw, rw`). **I- side**: `fence.i` for instruction stream coherence on non-coherent DIY systems; common application processors handle differently—see spec.

**AArch64**: `DMB`, `DSB`, `ISB`; load-acquire/store-release variants.

**x86**: `MFENCE`, `LFENCE`, `SFENCE` (some stronger guarantees pair with non-temporal stores/CLFLUSH families in system programming).

### 7.3 Per-ISA Consistency (Summary)

| ISA | Typical programmer stance |
|-----|---------------------------|
| **x86 / x86-64** | Stronger by default; still use atomics and fences correctly with SIMD/lock-free litmus; compiler barriers map to LOCK’d ops and fences. |
| **ARMv8** | Relaxed; **use C11/C++11 atomics** or explicit barriers; don’t rely on intuition from x86. |
| **RISC-V** | Relaxed; atomics via **A** extension; fences explicit. |

**Litmus tests** (e.g., **herd** models) exist to validate cross-ISA reasoning with formal memory models.

---

## 8. ISA Extensions and Evolution

### 8.1 How ISAs Evolve

Mechanisms:
- **Reserved encodings** become new opcodes.
- **Mandatory profile bumps** (RISC-V RVA*, ARM architecture updates).
- **CPUID / MIDR / mvendorid/marchid**-style discovery of features.
- **OS enablement** (save/restore new state on context switch: xsave/xrstor family, ARM SIMD/vector, RISC-V vector `vtype/vl`).

### 8.2 Vector / SIMD Timeline (Illustrative, Not Exhaustive)

```
  mid-90s: MMX (x86), MDMX/Paired-single (some MIPS)
     |
  SSE1/SSE2/SSE3/SSSE3/SSE4.x (x86) ----> AVX (256-bit YMM) ----> AVX2
     |                                              \
  ARM NEON (32-bit era)                               AVX-512 (512-bit ZMM + masks)
     |
  AArch64 NEON baseline in app CPUs
     |
  SVE/SVE2 (ARM scalable vectors)
     |
  RISC-V Vector (RVV) fixed/len-agnostic programming model
```

### 8.3 Virtualization Extensions

| Tech | Idea |
|------|------|
| **Intel VT-x** | Root vs non-root; VMCS controls traps; EPT extended page tables. |
| **AMD-V** | Similar root/guest with VMCB; nested paging (NPT). |
| **ARM** | EL2 two-stage translation; **VHE** optimizes host path. |
| **RISC-V H-extension** | HS/VS modes; hypervisor traps. |

### 8.4 Security Extensions (Examples)

| Feature | Idea |
|---------|------|
| **Intel SGX** | Enclaves (product lifecycle evolved; availability model changed by generation—consult current Intel docs). |
| **Intel CET** | Shadow stack and indirect branch tracking to mitigate ROP/JOP. |
| **ARM TrustZone** | Secure vs normal worlds; TZASC/TZPC peripherals. |
| **ARM Pointer Authentication (PAuth)** | Sign pointers; fault on tamper. |
| **RISC-V PMP** | Physical memory protection machine-mode regions for low-end control. |

---

## 9. Practical: Reading ISA Manuals

### 9.1 Intel / AMD x86-64

- **Intel 64 and IA-32 Architectures Software Developer’s Manuals (SDM)** multi-volume: Basic architecture, instruction set, system programming.
- **AMD APM** (Architecture Programmer’s Manual) complements vendor specifics.
- **How to read an entry**: mnemonic, **opcode map**, **instruction attributes** (64-bit mode validity, privilege), **operation pseudo-code**, **flags affected**, **exceptions**.
- **Encoding**: use **opcode bytes** tables + **ModR/M** appendix; watch **REX** and mandatory prefixes.

### 9.2 ARM

- **ARM Architecture Reference Manual (Armv8, for A-profile)** from Arm Ltd.
- Instruction **alphabetical** or **encoding tree** chapters; look for **alias** instructions (assembler mnemonics mapping to same encoding).
- **System register** access separate from AArch64 instruction descriptions.

### 9.3 RISC-V

- **Unprivileged ISA** + **Privileged Architecture** PDFs (RISC-V International).
- Descriptions specify **base** vs **extension**; ratification versions noted in frozen portions.
- **CSR** entries list fields and side effects.

### 9.4 Understanding Instruction Descriptions

Typical elements:
- **Assembly syntax** (operand order—AT&T vs Intel matters on x86).
- **Operands type** (reg/mem/imm).
- **Operation** section—algebraic and bitwise behavior.
- **Architectural state** updates: GPRs, flags, PC, CSRs/FPSR.
- **Constraints**: alignment, privilege, trap conditions.

### 9.5 Worked Example: Decoding a Real x86-64 Instruction Byte-by-Byte

**Machine bytes**: `48 83 C0 01`  
**Disassembly** (Intel syntax): `add rax, 1`

| Byte | Meaning |
|------|---------|
| `0x48` | **REX** prefix: `0100 1000` → `REX.W = 1` → default 64-bit operand size for qualifying instructions; other REX bits clear in this pattern. |
| `0x83` | Primary opcode: **Group 2** / immediate-8 form used for **ADD** with sign-extended byte immediate to r/m. |
| `0xC0` | **ModR/M**: `mod=11` (register mode), `reg=000` (ADD / opcode extension), `r/m=000` (rax in 64-bit register encoding for this group). |
| `0x01` | **Immediate**: 8-bit `0x01`, **sign-extended** to operand size (here 64-bit) → `+1`. |

```
  Byte stream walk:

  [48]       REX prefix activates 64-bit operand interpretation
     [83]    opcode family selects "alu imm8" path
        [C0] ModR/M says "destination is RAX, opcode ext ADD"
           [01] imm8 = +1
```

> **Caveat**: Full x86 decoding requires tables for all opcode maps, escape sequences (`0F xx`), `VEX/EVEX`, and legacy prefixes. The above is a clean pedagogical case.

### 9.6 Worked Example: RISC-V `addi` Skeleton

If `addi rd, rs1, imm` with opcode `OP-IMM` (`0010011`), `funct3=000` for **ADDI**:
- Fields: `imm[11:0]`, `rs1`, `funct3=000`, `rd`, `opcode`.

Example bit pattern (illustrative values only—compute exact bits when `rd/rs1/imm` fixed):

```
 imm[11:0]     rs1[4:0] 000 rd[4:0] 0010011
 +-----------+-------+---+-------+---------+
 |  12 bits  |  5    |3  |  5    |  7      |
 +-----------+-------+---+-------+---------+
```

Use the **RISC-V instruction listing** or an assembler (`llvm-objdump -d`) to verify encoding.

**Concrete word**: ` addi x5, x6, 3` assembles to **`0x00330293`** (RV32I/RV64I):

```
  imm[11:0]=3   rs1=x6=6  funct3=000  rd=x5=5  opcode=0010011
  000000000011  00110     000          00101     0010011
  \_____________/\_____/   \_/        \_____/   \______/
     bits 31:20   19:15    14:12      11:7       6:0
```

Check with: `riscv64-unknown-elf-as` / `llvm-mc` and `objdump -d`.

### 9.7 Worked Example: AArch64 `ADD` Register (Pattern)

Instructions like `ADD Xd, Xn, Xm` belong to `Add/Subtract (extended register)` or `(shifted register)` families depending on exact encoding; `sf=1` selects 64-bit. Consult the **A64 instruction encoding** diagram for **ADD (shifted register)**; fields identify `Rn`, `Rm`, `Rd`, shift amount (if any).

Verify with toolchain:

```text
aarch64-linux-gnu-as -c -o tmp.o file.s
aarch64-linux-gnu-objdump -d tmp.o
```

---

## Closing Notes

- **ISA is the contract**; **microarchitecture is the implementation**. Profiling teaches the latter; correctness for multi-threaded code requires the former’s memory model.
- **Prefer formal tools and vendor manuals** over folklore—especially when porting lock-free structures between x86, ARM, and RISC-V.
- **Extensions** move fast; **feature detection** and **OS support** are as important as raw instruction availability.

---

## References (Official Sources)

- Intel **64 and IA-32 Architectures Software Developer’s Manuals**.
- AMD **Architecture Programmer’s Manual**.
- Arm **ARM Architecture Reference Manual** (Armv8-A and later).
- **RISC-V International** specifications: Unprivileged ISA, Privileged Architecture, profiles, and extensions.
- MIPS Technologies / legacy **MIPS32/64 Architecture** manuals (for classic details).

*This tutorial is for education; always verify critical behavior against the ratified manuals for your target CPU and privilege level.*
