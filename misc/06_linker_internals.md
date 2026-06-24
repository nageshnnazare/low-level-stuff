# Linker Internals

This tutorial explains **how linkers work** at a conceptual and practical level: symbol resolution, relocation, static and dynamic linking, linker scripts, LTO, modern linker implementations, optimizations, and debugging. Examples assume **ELF** on Unix-like systems; many ideas transfer to **Mach-O** and **PE** with different names and details.

---

## Table of Contents

1. [What is a Linker?](#1-what-is-a-linker)
2. [The Linking Process Step by Step](#2-the-linking-process-step-by-step)
3. [Symbol Resolution](#3-symbol-resolution)
4. [Linker Scripts](#4-linker-scripts)
5. [Relocation](#5-relocation)
6. [Static Linking in Detail](#6-static-linking-in-detail)
7. [Dynamic Linking in Detail](#7-dynamic-linking-in-detail)
8. [Link-Time Optimization (LTO)](#8-link-time-optimization-lto)
9. [Modern Linkers](#9-modern-linkers)
10. [Linker Optimizations](#10-linker-optimizations)
11. [Debugging Linker Issues](#11-debugging-linker-issues)
12. [Practical Examples](#12-practical-examples)

---

## 1. What is a Linker?

### 1.1 The linker’s role in the build pipeline

A **compiler** (or assembler) turns one **translation unit** into a **relocatable object file** (`.o`). That file is almost never runnable by itself: it contains **machine code and data**, but **addresses are provisional**, **external references are unresolved**, and **pieces of the program still live in separate files**.

The **linker**’s job is to **combine** those pieces into a **single address space** (or a coherent shared library image) and produce:

- An **executable** you can load and run, or
- A **shared library** (`.so` / `.dylib`) loadable at runtime, or
- Occasionally another **relocatable** artifact (partial linking with `ld -r`, uncommon in app dev).

**At a high level, the linker:**

1. **Reads** object files and libraries.
2. **Resolves symbols** (function names, global variables) across inputs.
3. **Merges sections** (e.g. all `.text` together).
4. **Assigns final addresses** (layout).
5. **Patches** code and data (**relocations**) so instructions refer to the correct final addresses.
6. **Writes** the output ELF (or other format) with program headers, dynamic section, etc.

### 1.2 Pipeline diagram: source → executable

Simplified view of the usual C/C++ pipeline:

```
  +----------+     +-----------+     +------------+     +-------------+
  | Source   |     | Compiler  |     | Assembler  |     | Relocatable |
  | .c/.cpp  | --> | (cc1 etc) | --> | (as / gas) | --> | object .o   |
  +----------+     +-----------+     +------------+     +-------------+
                                                                |
                    multiple .o + libs (.a / .so)               |
                                                                v
                        +------------------------------------------------+
                        |                    LINKER (ld)                 |
                        |  resolve symbols, merge, relocate, emit image  |
                        +------------------------------------------------+
                                                |
                                                v
                        +------------------+------------------+
                        |                  |                  |
                        v                  v                  v
                 +-----------+    +--------------+   +--------------+
                 | Executable|    | Shared lib   |   | Partial link |
                 | (a.out)   |    | libfoo.so    |   | merged.o     |
                 +-----------+    +--------------+   +--------------+
```

**Key idea:** The assembler produces **fragments** with **holes** (relocatable fields). The linker **fills the holes** after it knows where everything landed.

### 1.3 Static linking vs dynamic linking (overview)

| Aspect | Static linking | Dynamic linking |
|--------|----------------|-----------------|
| **When library code is bound** | **Link time**: code from `.a` archives (or `.o`) is copied into the executable (or fully resolved at link for some models). | **Load/run time**: executable references `libfoo.so`; OS loader maps it and resolves symbols (with help from **PLT/GOT**). |
| **Artifact** | Larger executables; no runtime dependency on `libfoo.so` (if fully static). | Smaller executables; **shared** copy of library in RAM across processes. |
| **Updates** | Re-link to pick up library fixes. | Replace/update shared library (careful with **ABI** / **soname**). |
| **Startup** | No dynamic resolver work for that library. | Loader + **symbol lookup**; lazy binding can defer some work. |

**Static (typical):** `gcc main.o -ofoo libbar.a` — members of `libbar.a` are pulled in as needed.

**Dynamic (typical):** `gcc main.o -ofoo -lbar` — links against `libbar.so` (import symbols); **does not** embed `libbar`’s `.text` into `foo` (except via static archives if mixed).

```
Static mental model:
  main.o  +  libc pieces from libc.a  -->  one big blob in a.out

Dynamic mental model:
  main.o  +  import stubs (PLT)  -->  a.out still needs libc.so at runtime
                    |
                    +-------> loader maps libc.so, fixes GOT entries, etc.
```

---

## 2. The Linking Process Step by Step

### 2.1 Complete linking pipeline (conceptual)

```
+-----------------------------------------------------------------------------+
|                           LINKER INTERNAL PIPELINE                          |
+-----------------------------------------------------------------------------+
|  INPUTS:  foo.o  bar.o  baz.o  libcrt*.o  libm.a  libgcc.a  libfoo.so       |
+-----------------------------------------------------------------------------+
|  (0) Command-line / LTO plugin / emulation parsing                          |
|       - sysroot, arch emulation, -z flags, --gc-sections, etc.              |
+-----------------------------------+-----------------------------------------+
|  (1) SYMBOL RESOLUTION            |  Build global symbol table:             |
|       - scan definitions & refs   |  name -> {binding, section, value, ...} |
|       - link once rules           |  diagnose undefined / duplicate         |
+-----------------------------------+-----------------------------------------+
|  (2) SECTION MERGING              |  .text.foo + .text.bar + ... -> .text   |
|       - concatenate members       |  likewise .data, .rodata, .bss, ...     |
|       - COMDAT / ICF may collapse |  order controlled by linker script      |
+-----------------------------------+-----------------------------------------+
|  (3) ADDRESS ASSIGNMENT           |  VMA/LMA for each output section        |
|       (layout / relocation prep)  |  alignment, padding, memory regions     |
+-----------------------------------+-----------------------------------------+
|  (4) RELOCATION PATCHING          |  For each R_* relocation record:        |
|       - fix up code/data offsets  |  write final values into image bytes    |
+-----------------------------------+-----------------------------------------+
|  (5) OUTPUT GENERATION            |  ELF headers, program headers,          |
|       - optional .dynamic, symtab |  optional interpreter, build-id, map    |
+-----------------------------------------------------------------------------+
                                    |
                                    v
                             executable or .so
```

**VMA vs LMA:** **VMA** is the **runtime virtual address**; **LMA** is the **load address** (important in embedded ROM/flash scenarios).

### 2.2 Step 1: Symbol resolution

Each object file exports **defined** symbols and imports **undefined** symbols. The linker matches **undefined** references to **exactly one** compatible **definition** (per the platform’s rules), possibly from another `.o` or from a library archive member.

If a symbol stays undefined → **link error** (unless building a shared object that is allowed to have undefined imports).

### 2.3 Step 2: Section merging

Object files are **sliced into sections** (`.text`, `.data`, ...). The linker **concatenates** input sections into **output sections** according to **section flags** (allocate vs debug), **name**, and the **linker script**.

### 2.4 Step 3: Address assignment (relocation preparation)

Once output sections have **sizes**, the linker assigns **base addresses** respecting **alignment** and **memory region** constraints. Symbol values (`st_value`) become **final offsets** within the output section or absolute VMAs.

### 2.5 Step 4: Relocation patching

Relocations record **where** and **how** to patch. Example: “at offset 0x12 in `.text`, add the final address of `printf`”. The linker computes the **addend** and **target**, writes the patched encoding.

### 2.6 Step 5: Output generation

Emit **ELF file header**, **section header table**, **program headers** (segments for loading), optional **dynamic linking metadata** (`.dynsym`, `.rela.plt`, ...), and optionally a **symbol table** / **debug link**.

---

## 3. Symbol Resolution

### 3.1 Symbol types: defined, undefined, common

In ELF **relocatable** objects, symbols in `.symtab` include:

- **Defined (STB_GLOBAL/STB_WEAK, section != SHN_UNDEF):** symbol has storage in some section (e.g. function at `.text+0x40`).
- **Undefined (section SHN_UNDEF):** referenced but not defined in this file — **needs a definition** elsewhere.
- **Common** (**legacy** C tentative definitions: `int x;` at file scope without `extern`): appear as **STT_COMMON** / special **SHN_COMMON** handling. Modern GCC can use **`-fno-common`** to put tentatives in **`.bss`** instead — reduces surprises.

```
foo.c                          bar.c
-----                          -----
int x;        // tentative      int x = 3;  // strong definition (usually)

Linker may merge tentative common with a real definition or error if incompatible.
Prefer: use explicit "extern" + one definition, or -fno-common.
```

### 3.2 Strong vs weak symbols

- **Strong:** typical **global function** or **initialized** global variable. Multiple strong definitions of the same symbol → **multiple definition** error (ODR in C++ is stricter at compile time, but link still catches many issues).
- **Weak:** **Language/extension** features (e.g. `__attribute__((weak))` in GCC/Clang): allows **overriding** default stubs; if no strong symbol, the weak one is used.

**Intuition:**

```
strong main_sym   beats   weak default_impl
  |                           |
  +-------> linker picks main_sym
```

### 3.3 Symbol resolution rules (multiple definitions)

Rules vary slightly by toolchain and symbol type; typical ELF/GNU behavior:

1. **Exactly one strong global definition** for a given symbol name (C linkage) in the final link of an executable.
2. **Weak** can fill in if no strong exists.
3. **Local symbols** (`STB_LOCAL`) **do not participate** in **global** resolution — they can repeat in different files under the same name without conflict **as long as they are not exported**.

**C++ and mangling:** The linker sees **mangled** names for most C++ globals; **ODR** requires consistent definitions across TUs — violations can cause hard-to-debug behavior before link errors.

### 3.4 Archive / static library scanning algorithm

A **static archive** (`libfoo.a`) is a **collection of `.o` files** with a **symbol index** (from `ranlib`).

**Classic single-pass algorithm (conceptual):**

```
LINKER STATE:
  U = set of currently unresolved symbols (undefined but referenced)
  repeat until no progress:
      for each archive A in left-to-right order on command line:
          for each member M in A (often via index):
              if M defines any symbol in U:
                  extract M (link it in)
                  add M's undefined references to U
                  remove now-defined symbols from U
```

**Why this matters:** If **no unresolved symbol** references a member, that **`.o` inside `.a` is not pulled in**. Conversely, pulling in a member can **introduce new undefined symbols** that cause **later** libraries to satisfy them.

### 3.5 Why library order matters on the command line

Because of the **archive scan**, this fails:

```
gcc main.o -lfoo      # libfoo.a references symbols from libbar, but bar wasn't needed yet
                       # when libfoo was scanned → members not pulled → undefined refs
```

Common fix: **dependency order**:

```
gcc main.o -lfoo -lbar
```

Or repeat libraries:

```
gcc main.o -lfoo -lbar -lfoo
```

Better: use **`--start-group` / `--end-group`** (GNU ld) to **retry** archives until stable (slower).

### 3.6 Diagram: resolution across object files

```
  +--------+    undefined: printf         +--------+
  | main.o | ---------------------------> |        |
  | def:   |                              | libc   |
  |  main  |    undefined: helper         | pieces |
  +--------+ ---------------------------->|        |
       |                                  +--------+
       | undefined: helper                     ^
       v                                       |
  +--------+         defines helper            |
  | util.o |-----------------------------------+
  +--------+

Symbol table after merge might look like:

  printf   -> [shared object / libc definition]   (import or static)
  main     -> .text + 0x0 in main.o merged region
  helper   -> .text + 0x?? in util.o merged region
```

---

## 4. Linker Scripts

### 4.1 What linker scripts are and when you need them

A **linker script** tells `ld` **how to lay out** output sections in memory: **which input sections go where**, **alignment**, **VMA/LMA**, **entry point**, and **symbol aliases**.

**You need a custom script when:**

- **Bare metal / firmware:** flash vs RAM, vector table at fixed address, no OS loader.
- **OS kernel:** higher-half mapping, bootstrap sections, per-CPU data.
- **Fine control:** placing **hot code** in fast memory, **special segments**, overlay models.
- **Embedded ABI contracts:** ensuring `.bss` zero-init regions match startup code.

For typical hosted **Linux user apps**, the **default script** is enough.

### 4.2 Default linker script (`ld --verbose`)

```
# Typical inspection:
x86_64-linux-gnu-ld --verbose > default.ld
# or
gcc -Wl,--verbose ...   # may print linker invocation + script

The printed script shows:
  OUTPUT_FORMAT, ENTRY, MEMORY (if used), SECTIONS, ASSERTs, etc.
```

**Reading it** is educational: you see how `.text`, `.data`, `.bss` are formed from **Input Section wildcards** like `*(.text .text.*)`.

### 4.3 Script syntax highlights

| Construct | Role |
|-----------|------|
| `MEMORY { ... }` | Name regions (flash, ram) with **origin** and **length** |
| `SECTIONS { ... }` | Map **output sections** from **input sections**, set **VMA/LMA** |
| `ENTRY(symbol)` | Executable **entry** symbol (e.g. `_start`) |
| `PROVIDE(sym = expr)` | Define symbol only if references exist / as alias |
| `ALIGN(n)` | Align to **n** bytes (power of two usually) |
| `AT(addr)` | **LMA** for an output section (load address) |
| `KEEP(...)` | Prevent `--gc-sections` from dropping needed pieces (e.g. `KEEP(*(SORT_BY_NAME(.ctors)))`) |

**Wildcards:** `*(.text .text.*)` pulls all `.text` and `.text.*` input sections into the output section being defined.

### 4.4 Defining memory regions

```
MEMORY
{
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   (rwx): ORIGIN = 0x20000000, LENGTH = 128K
}
```

Sections placed only in **FLASH** vs **RAM** determine where the **startup code** must copy **initialized data** and **zero BSS**.

### 4.5 Placing sections precisely (embedded / kernel)

Typical goals:

- **Vector table / boot stub** at a **fixed** flash address.
- **`.text`** in **read-only executable** flash.
- **`.data` LMA** in flash, **VMA** in RAM (initialized variables copied before `main`).
- **`.bss`** in RAM, **zeroed** before `main`.
- **Kernel:** early **bootstrap** in identity-mapped low memory; later **higher-half** text.

### 4.6 Annotated embedded linker script example

```ld
/* Example ONLY — addresses fictional; adapt to your MCU/linker/emulation. */

OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
ENTRY(Reset_Handler)

MEMORY
{
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   (rwx): ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS
{
  /* Vector table and reset code must sit at the start of FLASH. */
  .isr_vector :
  {
    KEEP(*(.isr_vector)) /* KEEP: don’t drop with gc-sections */
  } > FLASH

  .text :
  {
    *(.text .text.*)
    *(.gnu.linkonce.t.*)
    . = ALIGN(4);
  } > FLASH

  .rodata :
  {
    *(.rodata .rodata.*)
    . = ALIGN(4);
  } > FLASH

  /*
   * Initialized data is STORED in FLASH after rodata (LMA),
   * but RUNS from RAM (VMA). Startup copies LMA -> VMA.
   */
  _sidata = LOADADDR(.data);
  .data : AT (ADDR(.rodata) + SIZEOF(.rodata))
  {
    _sdata = .;
    *(.data .data.*)
    . = ALIGN(4);
    _edata = .;
  } > RAM

  .bss :
  {
    _sbss = .;
    *(.bss .bss.*)
    *(COMMON)
    . = ALIGN(4);
    _ebss = .;
  } > RAM

  /* Stack can be modeled as symbols for startup.S / crt0 */
  _stack_end = ORIGIN(RAM) + LENGTH(RAM);
}

PROVIDE(end = .);
```

**Annotations:**

- **`ENTRY`:** CPU jumps here on reset (often sets stack pointer, copies `.data`, clears `.bss`, calls `SystemInit`, then `main`).
- **`LOADADDR(.data)` vs `ADDR(.data)`:** load vs virtual address split.
- **`PROVIDE`:** `end` is a traditional end-of-heap symbol used by some runtimes.

### 4.7 Annotated kernel-oriented linker script sketch

Kernel scripts are **highly platform-specific**; this illustrates **ideas** only:

```ld
OUTPUT_FORMAT(elf64-x86-64)
ENTRY(_start)

PHDRS
{
  headers PT_PHDR PHDRS;
  text    PT_LOAD;
  rodata  PT_LOAD;
  data    PT_LOAD;
}

SECTIONS
{
  . = KERNEL_VMA_BASE + bootstrap_size; /* e.g. higher-half mapping constant */

  .text : AT(ADDR(.text) - KERNEL_VMA_BASE + PHYS_LOAD_OFFSET)
  {
    *(.text.multiboot) /* example: early boot data */
    *(.text .text.*)
  } :text

  .rodata : {
    *(.rodata .rodata.*)
  } :rodata

  .data : {
    *(.data .data.*)
  } :data

  .bss : {
    *(.bss .bss.*)
    *(COMMON)
  } :data

  /DISCARD/ : {
    *(.comment)
  }
}
```

**Ideas:**

- **`PHDRS`** maps ELF **program headers** carefully for the bootloader/loader.
- **VMA vs AT(...)** separates **virtual** addresses the kernel executes at from **physical** placement during load.
- **`/DISCARD/`** strips unneeded sections.

---

## 5. Relocation

### 5.1 How the linker patches addresses

Relocations answer: **“Given this instruction encoding placeholder, compute the final field from symbol S after layout.”**

Each relocation record points to:

- **Offset** in the section to patch
- **Type** (`R_X86_64_PC32`, `R_X86_64_PLT32`, `R_ARM_CALL`, …)
- **Symbol** index
- Sometimes **addend** (in `.rela.*`)

The linker:

1. Determines **S**’s **final address** or **PLT slot** address.
2. Combines with **PC** or **base** per relocation type.
3. Writes the result; if it **does not fit** the instruction field → **relocation truncated to fit** error.

### 5.2 Position-dependent vs position-independent code

| Kind | Meaning |
|------|---------|
| **Position-dependent code (non-PIC)** | Assumes **fixed load address** (or at least link-time known layout). Simple absolute relocations possible. |
| **Position-independent code (PIC)** | Safe when mapped at **varying base addresses**. Uses **PC-relative** addressing and **indirection** via **GOT** for globals in shared libs. |

**Typical flags:**

- `-fno-pic` / default on some static exe builds: **non-PIC** `.o` allowed.
- `-fPIC`: emit PIC, required for **shared libraries** on many platforms.

### 5.3 PC-relative vs absolute relocations

```
PC-relative (concept):
  EA = PC + offset
  good for jumps/calls within range; position insensitive in many cases

Absolute:
  EA = absolute_address(symbol)
  breaks if loaded at unexpected base unless fixed or relocated by dynamic loader
```

**x86-64 small code model** often uses **PC-relative** relocations for `call` near GOT/PLT stubs.

### 5.4 GOT and PLT generation (dynamic linking)

**GOT (Global Offset Table):** table of **pointers**; for position-independent code, **data** and sometimes **code addresses** are loaded indirectly.

**PLT (Procedure Linkage Table):** small stubs for **external function calls**; enables **lazy binding** (resolve on first call).

**Rough picture:**

```
  call printf@plt  --->  PLT[n]  --->  (first time) resolver fixes GOT[n]
                         |                 |
                         v                 v
                    real printf      GOT stores real address later calls jump direct-ish
```

### 5.5 Before / after relocation patching (ASCII)

**Before link** (relocatable `main.o` `.text` excerpt — conceptual):

```
Address  Bytes / Concept
-------  -----------------------------------------
0x00     ... instructions ...
0x10     call 0x00000000    <-- relocation: fill with offset to printf's final target
0x16     ...
```

**After link** (executable — conceptual):

```
Address  Bytes / Concept
-------  -----------------------------------------
0x401010 call printf@plt    <-- patched to real displacement to PLT slot
...
PLT:     trampoline ...
GOT:     [runtime-filled pointers]
```

**Dynamic:** final **`printf` pointer** might be resolved at load or first call.

### 5.6 Relocation entries: `.rel` vs `.rela` and addends

On many 64-bit ELF platforms, relocations live in **`.rela.*`** sections: each record has an **explicit addend** field. Older **`.rel.*`** encodings sometimes **store the addend in the place being patched** (the **implicit addend**).

```
Conceptual RELA entry:
  offset: where in section to patch (byte offset)
  type:   R_* enumerator (ABI-defined meaning)
  symbol: index into symtab
  addend: signed integer combined per ABI formula
```

Inspect with:

```sh
readelf -r foo.o          # relocations per section
llvm-readobj -r foo.o     # LLVM equivalent
```

### 5.7 Example relocation types (x86-64, illustrative)

| Relocation | Typical meaning |
|------------|-----------------|
| `R_X86_64_PC32` | `S + A - P` — **32-bit** PC-relative (`P` = patch address) |
| `R_X86_64_64` | `S + A` — **64-bit** absolute |
| `R_X86_64_PLT32` | Like PC32 but target may be **PLT slot** for IFUNC / symbol preemption cases |
| `R_X86_64_GOTPCREL` | GOT entry address relative to `P` — for `-fPIC` global data access |

**Symbols:** `S` = symbol value after layout, `A` = addend, `P` = place being relocated.

**Why truncation happens:** `R_X86_64_PC32` only has **32 signed bits** of displacement. If the final target is **> 2GiB away** in the wrong sense for the chosen code model, the linker reports **relocation truncated to fit**. Fixes include **different code model**, **GOT indirection**, or **splitting** code.

### 5.8 PIC global data access (GOT) — mini diagram

```
fPIC code for:  return global_var;

  mov global_var@GOTPCREL(%rip), %rax   # load address of GOT slot into %rax
  mov (%rax), %eax                      # load value via pointer

GOT (in .got):
  +------------------+
  | ptr to global_var|
  +------------------+
```

The **relocation** on the `mov` fixes up **where the GOT slot lives** relative to `RIP`. The **dynamic linker** (or static link) fills the **slot** with the real address of `global_var`.

---

## 6. Static Linking in Detail

### 6.1 How static libraries (`.a`) work

An **archive** is **not** a **shared library**. It is a **bundle of `.o`** files. Linking with `libfoo.a`:

- Does **not** automatically include all members.
- Includes a member **only if** needed to satisfy **currently unresolved** symbols.

```
libfoo.a internals (conceptual):

  foo.o        bar.o        baz.o
  -----        -----        -----
  def: x()     def: y()     def: z()
  ref: y()     ref: z()

If only x() referenced from outside, may pull foo.o -> needs y -> pulls bar.o -> needs z -> pulls baz.o.
```

### 6.2 Archive scanning algorithm (step by step)

Detailed trace:

```
Initialize U with all undefined symbols from explicit .o files on cmd line

For each input token in order:
  If token is relocatable object (.o):
      Merge it
      Add its STB_GLOBAL/STB_WEAK undefined symbols to U
      Remove symbols it defines from U

  If token is archive (.a):
      Set progress = true
      While progress:
          progress = false
          For each member m in .a (using symtab index order):
              If m exports any symbol name in U:
                  Extract & link m (as if .o)
                  progress = true
                  Update U like a normal .o

After all inputs:
  If U not empty -> error undefined reference
```

### 6.3 `--whole-archive` and `--start-group` / `--end-group`

**`--whole-archive` (GNU ld):** Forces **every member** of archives until balanced with **`--no-whole-archive`** → use when **ctors** or **registration tables** would otherwise not be pulled.

**`--start-group` ... `--end-group`:** Archives between them are **re-scanned** until no new members or fixed point — fixes **circular** static dependencies at cost of **link time**.

```
gcc main.o -Wl,--start-group -lfoo -lbar -Wl,--end-group
```

### 6.4 Dead code elimination (`--gc-sections` + `-ffunction-sections -fdata-sections`)

Compiler flags:

- `-ffunction-sections`: each function → **own** `.text.functionname` section
- `-fdata-sections`: data similarly

Linker flag:

- `--gc-sections`: **drop unreferenced input sections** starting from **roots** (entry, `__start_*`, kept sections, `KEEP()` in script)

**Effect:** smaller binaries; need **`KEEP`** for introspected metadata (some registration schemes).

### 6.5 Section merging diagram (multiple `.o` → one executable)

```
foo.o                          bar.o
.text.foo          .rodata.foo            .text.bar      .data.bar
|     |              |                     |               |
+-----+--------------+---------------------+---------------+
                    MERGED OUTPUT (conceptual)
+----------------------------------------------------------+
| .text = [.text.foo][.text.bar][...]                      |
| .rodata = [.rodata.foo][...]                             |
| .data = [.data.bar][...]                                 |
| .bss = ...                                               |
+----------------------------------------------------------+
| <-- VMA grows with alignment padding between output secs |
+----------------------------------------------------------+
```

---

## 7. Dynamic Linking in Detail

### 7.1 Shared libraries (`.so` / `.dylib`)

An **ELF shared object** is a **partially linked** image with:

- **Exported dynamic symbol table** (`.dynsym`)
- **Relocation tables** for load-time fixups (`.rela.dyn`, `.rela.plt`)
- **`DT_NEEDED`**: dependencies on other shared objects
- Often **PIC** `.text` and **GOT** / **PLT**

**macOS dylib** differs in Mach-O structure but uses **similar concepts** (lazy symbol pointers, stubs).

### 7.2 `soname`, real name, linker name

Typical Linux triplet:

| Name | Example | Role |
|------|---------|------|
| **linker name** | `libfoo.so` | **`-lfoo`** usually searches `libfoo.so` symlink for compile/link |
| **soname** | `libfoo.so.1` | **DT_SONAME**: ABI compatibility name — dynamic loader records this dependency |
| **real name** | `libfoo.so.1.2.3` | **versioned file** on disk |

```
/usr/lib/
  libfoo.so       -> libfoo.so.1
  libfoo.so.1     -> libfoo.so.1.2.3
  libfoo.so.1.2.3    (actual file)
```

### 7.3 How the linker creates PLT/GOT entries

When building an **executable** against shared libs:

- Undefined function symbols → **PLT slots** (`printf@plt`) + **GOT** entries for **lazy** binding
- **Dynamic relocations** instruct the **runtime loader** (`ld.so`) how to fill GOT entries

When building a **shared object**:

- Similar, but **export** vs **import** roles differ; **preemptible** symbols may need special handling

### 7.4 Symbol versioning (`@@`, `@`)

GNU symbol versioning embeds **version requirements** in shared objects:

- **`symbol@@VER_1`**: **default** version used when linking
- **`symbol@VER_0`**: **old** version still present for older binaries

Loader matches **requested version** recorded in the **dependent** binary vs **definitions** in provider `libfoo.so`.

### 7.5 Version scripts (`--version-script`)

**Linker version scripts** control **visibility** of symbols in shared libraries:

```
VER_1 {
  global:
    foo;
    bar;
  local:
    *;
};
```

**Effects:**

- Only **listed globals** exported; others **local** (not subject to interposition)
- Define **ABI surface** explicitly

**Closing the world (common pattern):**

```ld
/* Export a tiny ABI; hide everything else */
{
  global:
    mylib_init;
    mylib_create_handle;
    mylib_destroy_handle;
    extern "C++" {
      mylib::PublicApi*;
    };
  local:
    *;
};
```

Pass to the linker when building `libmylib.so`:

```sh
cc -shared -fPIC -o libmylib.so $(objects) \
  -Wl,--version-script=mylib.map -Wl,--no-undefined
```

**Version nodes / inheritance (GNU style):**

```
LIBMYLIB_1_0 {
  global:
    foo;
  local:
    *;
};

LIBMYLIB_2_0 {
  global:
    bar;
  local:
    *;
} LIBMYLIB_1_0;   /* 2.0 inherits unresolved symbols semantics from 1.0 */
```

Exact syntax is **toolchain-specific**; always validate with `readelf -V libmylib.so` (**version definitions** / **needed versions**).

**ASCII: ABI surface before vs after a version script**

```
Before (default visibility = exported):
  .dynsym exposes:  foo bar helper_impl sneaky_internal ...
                           |
                           v
After version script (only foo bar global):
  .dynsym exposes:  foo bar
  others become local:  helper_impl sneaky_internal ...
```

### 7.6 Visibility control (`-fvisibility=hidden`, attributes)

Compiler **default visibility** affects dynamic symbol export:

- **`-fvisibility=hidden`:** most symbols **hidden** unless marked **default**
- **`__attribute__((visibility("default")))`** on API symbols

**Benefits:** fewer exported symbols → **faster linking/loading**, fewer **collision** issues, better **optimization** (compiler knows address not preempted).

### 7.7 PLT/GOT lazy binding diagram

```
First call to ext_func():

  caller
     |
     | call ext_func@PLT
     v
  PLT[n] stub:
     jmp *GOT[n]   -----> initially points to resolver stub
     push index
     jump PLT0 -> dynamic linker resolves ext_func address, writes GOT[n]

Later calls:

  PLT[n]:
     jmp *GOT[n]   -----> now points directly to ext_func in mapped shared lib
     (no resolver)
```

---

## 8. Link-Time Optimization (LTO)

### 8.1 What LTO is

**Link-Time Optimization** lets the compiler **see the whole program** (or large chunks) **at link time** and optimize **across translation units**: inlining, devirtualization, dead code elimination, constant propagation, etc.

Without LTO, each `.c` file is optimized **in isolation** before emitting object code.

### 8.2 Full LTO vs Thin LTO

| Mode | How it works | Pros / cons |
|------|--------------|-------------|
| **Full LTO** | Objects contain **GCC bytecode** or **LLVM bitcode**; linker invokes **compiler plugin** to optimize **whole IR** then codegen | Best cross-TU opts; **slow**, **memory-heavy** |
| **Thin LTO (LLVM)** | **Summary** per module; linker builds **global index**; **parallel** backend codegen + lightweight merging | **Much faster** than full; very good in practice |

### 8.3 Integration with the linker (plugin API)

**GCC:** `-flto` uses **liblto** / **`gcc` as linker wrapper**; `ld` with `--plugin /usr/lib/bfd-plugins/liblto_plugin.so`.

**Clang/LLVM:** `-flto=thin` or `-flto`; **lld** has strong ThinLTO integration; GNU ld also works with LLVM LTO plugin when configured.

```
gcc -flto -O2 a.c b.c -o prog

Objects may hold LLVM bitcode or GIMPLE LTO streams instead of only native .text
At link:
  linker loads plugin -> optimizes merged IR -> emits final machine code -> normal relocation
```

### 8.4 LTO pipeline diagram

```
 TU a.c --------\
                 +--> [IR LTO blobs in a.o, b.o]
 TU b.c --------/              |
                               v
                    +----------------------+
                    |  LINKER + PLUGIN     |
                    |  - load IR           |
                    |  - global analysis   |
                    |  - optimize          |
                    |  - codegen           |
                    +----------------------+
                               |
                               v
                    native .text/.data in output
                               |
                               v
                    optional second pass reloc/fixup
```

### 8.5 Benefits and trade-offs

**Benefits:**

- **Cross-TU inlining** and **better devirtualization**
- **Smaller/faster** code when duplication removed
- **Better DCE** of unused globals across files

**Trade-offs:**

- **Slower links**, especially full LTO
- **Harder debug** unless **fat LTO** / split DWARF strategies
- **Build system** must use **consistent flags** across all TUs
- Some **linker features** interact subtly with **LTO symbol visibility**

---

## 9. Modern Linkers

### 9.1 GNU ld (BFD linker)

Classic **GNU linker** using **BFD** library for multi-format support. **Ubiquitous**, **mature**, **flexible** linker scripts; **baseline** for many behaviors documented in Binutils manuals.

### 9.2 gold (Google’s linker)

**ELF-focused** faster linker (historically optional in binutils). **Different** internals than BFD ld; **not always 100%** script-compatible — check your version for edge cases. Maintenance/compatibility posture varies by distribution.

### 9.3 lld (LLVM’s linker)

**Part of LLVM**. Excellent **Clang** integration; very good **performance**; **Unix** (`ld.lld`) and **Windows** (`lld-link`) flavors. Often chosen in LLVM-centric projects.

### 9.4 mold

**Modern speed-oriented** linker (developed by Rui Ueyama; initially ELF-focused). Aims for **extremely fast** links on large C++ codebases; **compatible** with common GNU ld options in many workflows — verify for **your** exact flags/scripts.

### 9.5 Comparison table (rule-of-thumb; versions matter)

| Linker | Backend focus | Typical speed | Script compatibility | Notable |
|--------|---------------|---------------|----------------------|---------|
| **GNU ld (BFD)** | General/multi | Baseline | **Reference** implementation | Ubiquitous |
| **gold** | ELF | Faster than BFD ld | Mostly good | Varied maintenance story |
| **lld** | ELF/COFF/Mach-O flavors | Very fast | Good for common cases | Great LLVM integration |
| **mold** | ELF (scope evolving) | Often fastest | Strives for GNU ld CLI compat | Younger ecosystem |

**Compatibility caveat:** Obscure **linker script** features, **LTO plugins**, and **emulation** targets may differ — **always CI your real link line**.

---

## 10. Linker Optimizations

### 10.1 Identical Code Folding (ICF)

**ICF** merges **semantically identical functions** (safe equality of IR/bytes under analysis) **into one copy**, rewriting call sites.

**Flag (lld):** `--icf=all|safe|none` (names vary by linker; GCC uses `-Wl,--icf=all` with gold/lld where supported)

**Risk:** Breaks code that compares **function pointer addresses** for identity — **avoid relying on distinct addresses** of identical functions.

### 10.2 String merging

Read-only **string literals** that are equal may be **merged** in `.rodata` (compiler/linker cooperation). Reduces size; affects assumptions about **pointer identity** of string literals (rarely an issue).

### 10.3 COMDAT groups

**COMDAT** sections (e.g. **inline functions**, **templates**, **vtables**) carry **`GRP_COMDAT`** — **one definition wins** among duplicates across TUs.

```
 TU1: inline void f(){}
 TU2: inline void f(){}
 --> COMDAT group for f picks a single canonical copy in output
```

### 10.4 Relaxation

Linker (or **final** codegen) may **shorten** instruction sequences when targets **prove in-range**:

- Turn **indirect** jumps into **direct** if final address known and reachable
- **TLS** access sequences optimized on some arches

**Benefits:** smaller/faster code; **must preserve** semantics and alignment constraints.

### 10.5 Section garbage collection

**`--gc-sections`** removes unreachable sections given **roots**. Works best with **`-ffunction-sections -fdata-sections`**.

**Pitfalls:** dynamically loaded code, **registration** via sections, **Vulkan/D3D** SPIR patterns — may need **KEEP** or **attributes** to retain.

---

## 11. Debugging Linker Issues

### 11.1 Common errors and fixes

| Symptom | Typical causes | Fixes |
|---------|----------------|-------|
| **`undefined reference to ...`** | Missing `.o`/`.a` member; wrong library order; ABI mismatch | Add library; reorder; `--start-group`; ensure C vs C++ linkage (`extern "C"`); inspect `nm -u` |
| **`multiple definition of ...`** | Same strong symbol in two objects | Remove duplicate; fix `#include` implementation in headers; check **ODR**; unify weak/strong |
| **`relocation truncated to fit`** | Offset too large for instruction field; **wrong code model** | `-mcmodel=large` (or medium/small as appropriate); PIC vs non-PIC mix; split huge binaries |
| **`cannot find -lfoo`** | Search path / missing dev package | `-L`, `LIBRARY_PATH`, install `-dev` package, check `ldconfig` for runtime `.so` vs link for linker name |

### 11.2 Using `-Wl,--trace` and `-Wl,-Map=file.map`

```
gcc a.o b.o -Wl,--trace -Wl,-Map=link.map -o prog

--trace: prints which input files/objects/archives are opened
Map file: final symbol addresses, section sizes, sometimes discard lists (gc-sections)
```

### 11.3 Reading map files (what to look for)

Sections to inspect:

- **Memory configuration** (if echoed)
- **Output section** layout: **VMA**, **size**, **symbols** sorted by address
- **Discarded input sections** (if logged)
- **Cross references** (linker-dependent)

**Use map files to answer:**

- “**Where** did `my_big_array` end up?”
- “**Why** is `.text` huge?” (sometimes **template explosion** visible as many symbols)
- “**Which** archive member defined `symbol`?”

---

## 12. Practical Examples

### 12.1 Linking a multi-file C project manually

Files:

```
main.c   -> main.o
utils.c  -> utils.o
```

Commands:

```sh
cc -c main.c -o main.o
cc -c utils.c -o utils.o
cc main.o utils.o -o app
# equivalent explicit ld (platform paths fictional — prefer cc as driver):
# ld -dynamic-linker /lib64/ld-linux-x86-64.so.2 \
#    /usr/lib/Scrt1.o /usr/lib/crti.o main.o utils.o \
#    -lc /usr/lib/crtn.o -o app
```

**Why use `cc` as driver:** It supplies **crt** objects, **default libs**, **pie** flags, and **target-specific** paths correctly.

### 12.2 Creating and using a shared library

```sh
# Build PIC objects
cc -fPIC -c foo.c -o foo.o

# Link shared library with soname
cc -shared -Wl,-soname,libfoo.so.1 -o libfoo.so.1.0.0 foo.o
ln -s libfoo.so.1.0.0 libfoo.so.1
ln -s libfoo.so.1 libfoo.so

# Link executable against it
cc main.o -L. -lfoo -Wl,-rpath,'$ORIGIN' -o prog
```

**`$ORIGIN` in rpath:** locates ** beside the executable** — useful for **relocatable** installs.

### 12.3 Custom linker script for bare metal (workflow)

1. Write `mem.ld` (MEMORY + SECTIONS) matching **MCU** memory.
2. Supply startup (**`.S`**) that uses symbols **`_sidata`, `_sdata`, `_edata`, `_sbss`, `_ebss`** consistent with script.
3. Link:

```sh
arm-none-eabi-gcc -T mem.ld -nostartfiles -nostdlib \
  startup.o main.o -lgcc -o firmware.elf
```

4. Convert to **hex/bin** with `objcopy` as your flash tool requires.

### 12.4 Analyzing a linker map file (mini checklist)

1. **Search** for your symbol (`grep` the `.map`).
2. Note **VMA** — matches **debugger** PC for functions.
3. Compare **`.text`** total to **`-Wl,--print-gc-sections`** output if sizes surprise.
4. For **duplicate** symbols, map often shows **file paths** of contributors.

### 12.5 Annotated map file excerpt (conceptual)

GNU ld **`-Map=out.map`** output is **not** standardized, but often resembles:

```
Memory Configuration   # if MEMORY {} used in script (embedded)

Name             Origin             Length             Attributes
FLASH            0x08000000         0x00080000         xr
RAM              0x20000000         0x00020000         xrw

SECTION SUMMARY      # sizes per output section
.text                0x08000100     0x1234
.rodata              0x08001334     0x0560
.data                0x20000000     0x0010  load address 0x08001894
.bss                 0x20000010     0x0040

.text section layout
 .text.main          0x08000100     0x40   main.o
 .text.util_fun      0x08000140     0x30   util.o
                     0x08000170            *fill*   # alignment padding
 .text.helpers       0x08000180     0x100  libstuff.a(stuff.o)

CROSS REFERENCE TABLE   # optional; shows who references whom
symbol              file
util_fun            main.o
```

**How to use this:**

- **`load address` vs VMA** on `.data` confirms **ROM → RAM** copy layout (embedded).
- **`fill`** lines show **alignment slack** — big fills suggest over-alignment or section ordering issues.
- **Per-object contributions** help attribute **code bloat** to a specific `.o` or archive member.
- If **`--gc-sections`** is on, some maps list **discarded** input sections — grep for `Discarded` or `Removing`.

---

## Further Reading

- **ELF specification** (Tool Interface Standard) — program headers, relocations, dynamic linking.
- **GNU ld manual** (Linker scripts, emulation, options).
- **`man ld.so`** — runtime loader behavior.
- **Agner Fog’s optimization manuals** — code models and addressing (architecture-specific).
- Your platform’s **ABI** supplement (e.g. System V AMD64 ABI) for PIC, TLS, and relocation types.

---

*End of document.*
