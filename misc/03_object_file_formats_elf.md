# Object File Formats: A Deep Dive into ELF

This tutorial explains what **object files** are, how they evolved across operating systems, and how **ELF** (Executable and Linkable Format) structures programs for linking and execution. It assumes familiarity with C compilation (`gcc`, `ld`) and basic binary / memory concepts.

> **Platform note:** Sample `readelf` / `objdump` transcripts in Section 9 are **representative of GNU/Linux x86-64**. On macOS, `/bin/*` binaries are **Mach-O**, and `readelf` is often absent unless you install GNU binutils (e.g. via Homebrew) or use a Linux VM/container.

---

## 1. Object File Formats Overview

### 1.1 What object files are and why they exist

When you compile a source file, the compiler does **not** usually produce a finished runnable program in one step (unless you force a single translation unit and link statically in a simplistic way). Instead:

1. **Preprocessing** expands macros and includes.
2. **Compilation** turns C/C++ into **assembly**, then **machine code** for one compilation unit (`.c` → `.o`).
3. **Assembly** may run as a separate pass or be internal to the compiler front end.
4. **Linking** combines `.o` files and libraries, resolves symbols, applies relocations, and emits the final **executable** or **shared library**.

An **object file** is the on-disk container for machine code and **metadata** the linker and loader need: symbols, relocations, section boundaries, debug info, dynamic linking tables, etc. Without a standard format, every toolchain would reinvent brittle custom layouts.

Object files exist to:

- **Separate compilation:** Build many `.o` files in parallel; link once.
- **Libraries:** Ship reusable code as **static archives** (`libfoo.a`: many `.o` members) or **shared objects** (`libfoo.so`).
- **Tooling:** Debuggers, profilers, and analyzers read structured symbol and unwind data.
- **OS loaders:** The kernel (or dynamic linker) maps **segments** into memory using program headers.

### 1.2 History: a.out, COFF, PE/COFF, Mach-O, ELF

| Era / system        | Format   | Notes |
|---------------------|----------|--------|
| Early Unix          | **a.out** | Simple; limited extensibility; name “assembler output” stuck even for linked executables. |
| PDP-11 / V7+ Unix   | a.out variants | Magic numbers, tiny header; hard to add features (shared libraries, complex sections). |
| System V / embedded | **COFF** | Section table + relocation entries; more structure than a.out. |
| Windows (32/64-bit) | **PE** (Portable Executable), **PE/COFF** | COFF-derived; PE wraps COFF with DOS stub, rich optional header, resources. |
| Apple               | **Mach-O** | Fat (multi-arch) binaries; load commands instead of ELF program headers; different linking model. |
| Most Unix/Linux, BSDs, many embedded | **ELF** | Flexible **section** (linking) and **segment** (loading) views; dominates Linux and many non-Windows platforms. |

**ELF** was introduced in the late 1980s / early 1990s (System V ABI, later Linux adoption). It cleanly separates:

- The **linking view** (sections, section header table).
- The **execution view** (segments, program header table).

### 1.3 Comparison table across operating systems

| Aspect | ELF (Linux, *BSD, many embedded) | PE/COFF (Windows) | Mach-O (macOS, *OS) |
|--------|----------------------------------|-------------------|----------------------|
| Typical extension | `.o`, `.so`, no extension for exe | `.obj`, `.dll`, `.exe` | `.o`, `.dylib`, Mach-O binary |
| Magic / identification | `7f ELF` (`0x7F 'E' 'L' 'F'`) | `MZ` + PE signature `PE\0\0` | `0xFE 0xED 0xFA 0xCE` (or `0xCE 0xFA 0xED 0xFE`) |
| Load metadata | Program headers (segments) | Optional header + section alignment | Load commands |
| Shared libs | `ET_DYN` + `.dynamic`, PLT/GOT | Import Address Table, delay load | dyld, dylib, indirect symbols |
| Debug / unwind | `.debug_*`, `.eh_frame` | PDB + PE debug dirs | DWARF + compact unwind |
| Position independence | PIE: `ET_DYN` + PIC | ASLR + relocations | PIE by default on Apple platforms |

---

## 2. ELF Format In Depth

### 2.1 Two views of the same file

```
                    ELF FILE ON DISK
    +============================================================+
    |                    ELF HEADER                              |
    +============================================================+
    |  LINKING VIEW (linker, readelf -S)  |  EXEC VIEW (loader)  |
    |  Section Header Table  +  Sections  |  Program Headers     |
    |  .text .data .bss ...               |  LOAD segments...    |
    +=====================================+======================+
```

- **Sections** (with **section header table**, SHT) describe how **linkers** merge code/data and apply relocations.
- **Segments** (with **program header table**, PHT) describe how the **loader** maps file regions into virtual memory.

A single byte range may appear in both a section and a segment—mappers tie them together by file offset and virtual address.

### 2.2 ASCII diagram: high-level ELF file layout (typical executable)

```
  File offset
  0x0000   +------------------+
           |   ELF Header     |  <- describes class (32/64), endianness,
           +------------------+     where section & program headers are
           |  Program Header  |  <- often immediately after ELF header
           |  Table (PHT)     |     (or later; e_phoff points here)
           +------------------+
           |   ... gap ...    |  <- may be zero-filled / alignment
           +------------------+
  e_shoff  | Section Header   |
           | Table (SHT)      |  <- often near END of file
           +------------------+
           | .interp          |
           | .note.*          |
           | .hash / .gnu.hash|
           | .dynsym, .dynstr |
           | .rela.plt etc.   |
           | .init, .plt,     |
           | .text, .fini     |
           | .rodata          |
           | .eh_frame*       |
           | .init_array ...  |
           | .dynamic         |
           | .got*, .data*    |
           | .bss             |
           | [debug sections] |
           +------------------+
```

Not every file has all pieces; **relocatable objects** (`.o`) have **no program headers** (`e_phnum == 0`), only sections.

### 2.3 ELF Header — concepts

Key fields (conceptual):

| Field | Role |
|-------|------|
| **e_ident** | Magic, class (32/64), endianness, ELF version, OS/ABI, ABI version |
| **e_type** | `ET_REL`, `ET_EXEC`, `ET_DYN`, `ET_CORE`, … |
| **e_machine** | ISA: e.g. `EM_386`, `EM_X86_64`, `EM_ARM`, `EM_AARCH64`, … |
| **e_version** | Object file version (usually `EV_CURRENT`) |
| **e_entry** | Virtual address where execution starts (`main` is not this; often `_start` / CRT) |
| **e_phoff** | File offset of program header table |
| **e_shoff** | File offset of section header table |
| **e_flags** | Processor-specific flags |
| **e_ehsize** | Size of this ELF header |
| **e_phentsize** | Size of one program header entry |
| **e_phnum** | Number of program headers |
| **e_shentsize** | Size of one section header entry |
| **e_shnum** | Number of section headers |
| **e_shstrndx** | Section header index of **section name string table** (`.shstrtab`) |

**e_ident[EI_MAG0..3]:** `0x7F`, `'E'`, `'L'`, `'F'`.

**e_ident[EI_CLASS]:** `ELFCLASS32 (1)` or `ELFCLASS64 (2)`.

**e_ident[EI_DATA]:** `ELFDATA2LSB (1)` little-endian or `ELFDATA2MSB (2)` big-endian.

**e_ident[EI_VERSION]:** Usually `EV_CURRENT (1)`.

**e_ident[EI_OSABI] / EI_ABIVERSION:** OS/ABI tagging (often `ELFOSABI_SYSV` on Linux userland with `0` ABI version).

### 2.4 Exact byte layout and field sizes

#### ELF header — **32-bit** (`Elf32_Ehdr`)

```
Byte offset   Field            Size      Description
-----------   -----            ----      -----------
0x00          e_ident          16        Magic, class, data, version, pad, OSABI, ABI ver
0x10          e_type           2         ET_* 
0x12          e_machine        2         EM_*
0x14          e_version        4         EV_*
0x18          e_entry          4         Entry point (VA)
0x1C          e_phoff          4         Program header table file offset
0x20          e_shoff          4         Section header table file offset
0x24          e_flags          4         Processor-specific
0x28          e_ehsize         2         sizeof(Elf32_Ehdr) typically 52
0x2A          e_phentsize      2         One Phdr size
0x2C          e_phnum          2         Number of Phdrs
0x2E          e_shentsize      2         One Shdr size
0x30          e_shnum          2         Number of Shdrs
0x32          e_shstrndx       2         .shstrtab section index
              TOTAL            52 bytes
```

#### ELF header — **64-bit** (`Elf64_Ehdr`)

```
Byte offset   Field            Size      Description
-----------   -----            ----      -----------
0x00          e_ident          16        (same semantics as 32-bit)
0x10          e_type           2
0x12          e_machine        2
0x14          e_version        4
0x18          e_entry          8         Entry point (VA)
0x20          e_phoff          8         Program header offset
0x28          e_shoff          8         Section header offset
0x30          e_flags          4
0x34          e_ehsize         2         sizeof(Elf64_Ehdr) typically 64
0x36          e_phentsize      2
0x38          e_phnum          2
0x3A          e_shentsize      2
0x3C          e_shnum          2
0x3E          e_shstrndx       2
              TOTAL            64 bytes
```

---

## 3. ELF Sections

### 3.1 Section Header Table (SHT)

Each **section header** (`Elf32_Shdr` / `Elf64_Shdr`) describes one contiguous region of the file.

#### 64-bit section header entry (`Elf64_Shdr`) — layout

```
Offset  Field           Size    Purpose
------  -----           ----    -------
0x00    sh_name         4       Offset into .shstrtab of NUL-terminated name
0x04    sh_type         4       SHT_PROGBITS, SHT_SYMTAB, …
0x08    sh_flags        8       SHF_WRITE, SHF_ALLOC, SHF_EXECINSTR, …
0x10    sh_addr         8       Virtual address (0 if not loaded)
0x18    sh_offset       8       File offset of section bytes
0x20    sh_size         8       Size in bytes
0x28    sh_link         4       Link to another section (semantics depend on type)
0x2C    sh_info         4       Extra info (e.g. symbol table for reloc section)
0x30    sh_addralign    8       Alignment requirement
0x38    sh_entsize      8       Size of entries if table (symtab, rela, …)
        TOTAL           64 bytes
```

#### ASCII: sections as file slices

```
File
address
   |   Section A (e.g. .text)
   |   [================================)
   |              Section B (.rodata)
   |                    [======================)
   v
```

The **section header table** is itself not loaded at runtime unless some odd tooling maps it; normal executables mark SHT as “not loaded”—executables run from **segments**, not section table entries.

### 3.2 Important sections (detailed)

#### `.text`

- **Content:** Executable machine instructions.
- **Typical flags:** `SHF_ALLOC | SHF_EXECINSTR` (not writable in hardened binaries).
- **Role:** The main code the CPU runs (plus compiler-generated helpers).

#### `.data`

- **Content:** Initialized **writable** global/static variables (`int x = 5;`).
- **Flags:** `SHF_ALLOC | SHF_WRITE`.
- **Role:** Lives in a read-write segment (`RW`); must be copied from a file because initial values are stored on disk.

#### `.bss`

- **Content:** **Uninitialized** (or zero-initialized) writable globals (`static int y;` or C `TBSS`).
- **Flags:** Often `SHF_ALLOC | SHF_WRITE`; **SHT_NOBITS** — occupies **no disk bytes** beyond the metadata; loader **zero-fills** `sh_size` bytes in memory.
- **Role:** Saves executable size.

#### `.rodata`

- **Content:** Read-only literals: string constants, `const` data, jump tables, vtables (in C++ often split further).
- **Flags:** `SHF_ALLOC`; not writable.
- **Role:** Can be mapped read-only → helps against some memory corruption exploits.

#### `.symtab` and `.dynsym`

- **`.symtab`:** Full **symbol table** for **static** linking / debugging (often stripped in release binaries).
- **`.dynsym`:** **Dynamic** symbol subset — symbols visible for dynamic linking; kept in production shared objects and executables using `libc.so`, etc.

Each **symbol table** pairs with a string table: **`.strtab`** for `.symtab`, **`.dynstr`** for `.dynsym`.

#### `.strtab` and `.dynstr`

- **Content:** Dense pack of NUL-terminated strings; symbol names, section names for some tools, etc.
- **Referenced by:** `st_name` fields in symbol entries as byte offsets.

#### `.rel*` and `.rela*` (relocations)

- **`.rel`:** Relocation entries **without** explicit addend (addend is in the patch location for some types).
- **`.rela`:** Relocation entries **with** explicit `r_addend` field **in the reloc record** — x86-64 uses **Rela** heavily (e.g. `.rela.dyn`, `.rela.plt`).

Sections link to the symbol table they refer to via `sh_link` and related `sh_info` (which section is being relocated).

#### `.plt` and `.got` / `.got.plt`

- **GOT (Global Offset Table):** Holds **addresses** of globals / external functions after dynamic resolution (or fixed-up at load time).
- **PLT (Procedure Linkage Table):** Small code stubs; each external call typically goes **PLT entry → resolver / resolved target**. Enables **lazy binding** for functions.
- **`.got.plt`:** GOT entries used with PLT (classic GNU ld layout).

#### `.init`, `.fini`, `.init_array`, `.fini_array`

- **`.init` / `.fini`:** Legacy **function** sections (deprecated style on modern glibc but still seen).
- **`.init_array` / `.fini_array`:** Arrays of function pointers run **before `main`** and **at exit** / unload — C++ static constructors, `__attribute__((constructor))`.

#### `.eh_frame`

- **Content:** **DWARF-based** exception handling / stack-unwind tables (Itanium C++ ABI / GCC).
- **Role:** `throw` unwinding, backtraces; cooperates with `.gcc_except_table` (LSDA) when C++ exceptions used.

#### `.note.*` (e.g. `.note.gnu.build-id`)

- **Content:** **Note records** (name, type, descriptor) — e.g. **build ID** for debug-symbol servers (`debuginfod`), ABI tags, vendor info.
- **Program header:** Often exposed as **`PT_NOTE`** segment.

#### `.interp`

- **Content:** Path to **dynamic linker** as a string, e.g. `/lib64/ld-linux-x86-64.so.2`.
- **Program header:** Mapped via **`PT_INTERP`**; kernel uses it to start the interpreter for dynamically linked executables.

#### `.dynamic`

- **Content:** Array of **`Dyn` entries** (`DT_NEEDED`, `DT_PLTGOT`, `DT_SYMTAB`, …).
- **Role:** The **directory** for the dynamic linker: what libraries to load, where GOT/PLT/tables are, flags like `BIND_NOW`, etc.
- **Segment:** Usually inside a `PT_LOAD` or paired with **`PT_DYNAMIC`**.

#### `.hash` and `.gnu.hash`

- **`.hash`:** Classic **SysV hash** bucket chains for dynamic symbol lookup (`DT_HASH`).
- **`.gnu.hash`:** GNU’s faster **Bloom-filter-augmented** hash (`DT_GNU_HASH`) — can skip probing for absent symbols cheaply.

---

## 4. ELF Segments (Program Headers)

### 4.1 Sections vs segments

```
  LINKER VIEW                           LOADER / RUNTIME VIEW
  (sections)                           (segments)

  +--------+--------+--------+          +========================+
  | .text  | .rodata| .init  |   --->   | PT_LOAD (R X)          |
  +--------+--------+--------+          |  maps .text,.rodata,...|
  | .data  | .bss   |        |   --->   | PT_LOAD (RW)           |
  +--------+--------+--------+          |  maps .data + zero bss |

  Multiple sections ONE segment         One segment may contain
  merged if same flags/alignment        many sections' file bytes
```

**Program header** (`Elf64_Phdr` example fields):

```
p_type    : LOAD, DYNAMIC, INTERP, ...
p_flags   : PF_X, PF_W, PF_R (execute, write, read)
p_offset  : Where in file this segment starts
p_vaddr   : Preferred virtual address
p_paddr   : Physical addr (often ignored on many OSes)
p_filesz  : Bytes from file to map
p_memsz   : Bytes in memory (may be > filesz → BSS-like zero fill)
p_align   : Alignment
```

### 4.2 Common `p_type` values

| Type | Name | Role |
|------|------|------|
| `PT_LOAD` | Loadable segment | Mapped into memory; main code/data. |
| `PT_DYNAMIC` | Dynamic linking | Points to `.dynamic` array metadata (often redundant with address in LOAD). |
| `PT_INTERP` | Interpreter | Path string for dynamic linker. |
| `PT_NOTE` | Notes | Build ID, ABI notes, etc. |
| `PT_GNU_STACK` | GNU stack | Declares whether stack is executable (`NX` bit behavior). |
| `PT_GNU_RELRO` | Read-only after reloc | After dynamic linker finishes relocations, can mark GOT etc. read-only. |
| `PT_TLS` | Thread-local | Describes TLS template for `__thread` / TLS variables. |

### 4.3 How the kernel maps segments (simplified Linux)

```
execve("/bin/ls", ...)
    |
    v
Kernel reads ELF header + program headers
    |
    +-- For each PT_LOAD:
    |       mmap-like: file range [p_offset, p_offset+p_filesz)
    |                  -> VA [p_vaddr, p_vaddr+p_memsz)
    |       Protections from p_flags (NX on modern x86-64)
    |       Zero-fill (anonymous) for (p_memsz - p_filesz) tail
    |
    +-- If PT_INTERP: load interpreter (ld.so) too
    |
    v
Jump to interpreter entry OR e_entry (static exe)
    |
    v
Interpreter loads NEEDED libs, applies relocations, then jumps to program CRT / _start
```

Address space layout may use **PIE** (`ET_DYN` executable) with **ASLR**; then **base address** is randomized at load time while preserving **relative** layout between segments.

---

## 5. Symbol Tables

### 5.1 Symbol entry structure (64-bit `Elf64_Sym`)

```
Offset  Field      Size   Meaning
------  -----      ----   -------
0x00    st_name    4      Offset in string table (.strtab / .dynstr)
0x04    st_info    1      Binding + type packed (see macros)
0x05    st_other   1      Visibility + unused bits
0x06    st_shndx   2      Section index or special (SHN_UNDEF, ABS, COMMON, …)
0x08    st_value   8      Address / offset depending on context
0x10    st_size    8      Size of symbol (bytes) for objects/functions
Total          24 bytes
```

**st_value meanings:**

- In **relocatable** `.o`: often **offset** within section `st_shndx`.
- In **executable/DSO** (after link): usually **virtual address**.
- **`SHN_COMMON`:** alignment requirements for tentative definitions.

`st_info` unpacking:

- **Binding:** `STB_LOCAL`, `STB_GLOBAL`, `STB_WEAK`.
- **Type:** `STT_NOTYPE`, `STT_OBJECT`, `STT_FUNC`, `STT_SECTION`, `STT_FILE`, …

### 5.2 Symbol binding

| Binding | Meaning |
|---------|---------|
| **LOCAL** | Not visible outside the defining `.o` (after link: name may be discarded / not exported). |
| **GLOBAL** | Single definition across link units; external linkage by default in C. |
| **WEAK** | May be overridden by a strong **GLOBAL** symbol at link time; useful for optional stubs. |

### 5.3 Symbol types (selected)

| Type | Meaning |
|------|---------|
| **NOTYPE** | Unspecified (labels, aliases). |
| **OBJECT** | Data symbol. |
| **FUNC** | Function code. |
| **SECTION** | Relates to a section (debug / special). |
| **FILE** | Source file name marker (debugging / STT_FILE). |

### 5.4 Symbol visibility (`STV_*`)

| Visibility | Effect |
|------------|--------|
| **DEFAULT** | Normal export rules; interposable unless bindings prevent. |
| **INTERNAL** | Special processor-specific / reserved semantics in some ABIs. |
| **HIDDEN** | Not exported to other modules; not subject to **preempt** by another DSO. |
| **PROTECTED** | Defined in this DSO **must** resolve **within** this DSO (not preempted by external symbol of same name). |

### 5.5 Version symbols

Shared objects may use **symbol versioning** (`@@`, `@`) via **GNU version sections** (`.gnu.version`, `.gnu.version_r`). The dynamic linker picks the correct **default** or **tagged** version when multiple definitions exist across time (`GLIBC_2.2.5` vs `GLIBC_2.14`, etc.).

---

## 6. Relocations

### 6.1 Why relocations exist

A `.o` file is compiled **before** final addresses exist:

- Code may reference **globals** at unknown addresses.
- Calls to **external functions** need patch sites.
- **PIC** code may need **GOT-relative** or **PLT-relative** encodings.

The assembler emits **placeholder** bits + **relocation records** telling the linker (or dynamic linker) how to **patch** the code/data once addresses are known.

### 6.2 Relocation entry structure

**Rel** (`Elf64_Rel`) — **16 bytes** on x86-64:

```
r_offset : 8  (where to patch — offset in section / VA depending on stage)
r_info   : 8  Encodes (sym_index, type) via ELF64_R_SYM / ELF64_R_TYPE
```

**Rela** (`Elf64_Rela`) — **24 bytes**:

```
r_offset : 8
r_info   : 8
r_addend : 8  Explicit addend (adds to computed value)
```

On **x86-64**, **`r_info`** layout:

- Lower **32 bits:** `ELF64_R_TYPE`
- Upper **32 bits:** `ELF64_R_SYM` (symbol table index)

### 6.3 Common x86-64 relocation types (examples)

| Type | Typical use |
|------|-------------|
| `R_X86_64_64` | Full 64-bit absolute address (often in `.data` pointers). |
| `R_X86_64_PC32` | 32-bit PC-relative (from end of patched insn / ABI-defined PC). |
| `R_X86_64_PLT32` | PLT-slot-relative 32-bit (compiler emits `call foo@plt`). |
| `R_X86_64_GOTPCREL` | Offset to GOT entry, PC-relative. |
| `R_X86_64_GLOB_DAT` | Set GOT slot to symbol VA (dynamic). |
| `R_X86_64_JUMP_SLOT` | PLT/GOT lazy pointer. |
| `R_X86_64_RELATIVE` | Base + addend (no symbol) — common for PIE. |
| `R_X86_64_32` / `_32S` | Truncated absolute (careful with PIE / overflow). |

Exact numeric values appear in `<elf.h>` (`R_X86_64_*` enum).

### 6.4 ASCII: relocation patching process

```
  BEFORE LINK (in .o)

  .text:  mov 0x0(%rip), %rax     ; placeholder, R_X86_64_GOTPCREL on stdout@@GLIBC
          ...
  reloc:  offset -> that insn
          sym: stdout
          type: GOTPCREL
          addend: -4  (example)

  LINKER computes:
      address of GOT entry G for stdout
      PC_after_instruction = ...
      delta = G - PC_after + addend
      PATCH 32-bit immediate / disp

  AFTER LINK

  .text:  mov correct_disp(%rip), %rax
```

Dynamic linking repeats a similar idea at **load time** for symbols resolved from other DSOs.

---

## 7. Dynamic Linking

### 7.1 How shared libraries work

1. **Executable** lists **`DT_NEEDED`** entries in `.dynamic` (`libc.so.6`, …).
2. **Dynamic linker** (`ld.so`) maps those **ET_DYN** files into the address space.
3. **Symbol lookup** resolves undefined symbols using **search scope** (executable, then libs in order).
4. **Relocations** in **`JMPREL` / `.rela.plt` / `.rela.dyn`** patch GOT / code.
5. **Constructors** run (`INIT` / `init_array`).

### 7.2 PLT / GOT lazy binding — conceptual steps

Assume **lazy** binding (default): GOT slots for functions start pointing to resolver stubs.

```
Call site in main:
    call printf@plt
         |
         v
+----------------------------+
| .plt entry for printf      |
|   jmp *GOT[n]  -----------+---> first time: points to .plt resolver push n / jmp
+---------------------------+     later: points to real printf in libc
         |
         (first call)
         v
+---------------------------+
| Dynamic linker resolver   |
|   - finds real printf     |
|   - writes GOT[n]         |
|   - jumps to printf       |
+---------------------------+
```

**ASCII timeline**

```
Time T0 (after load, before first printf call):

   GOT[n]  =  address of PLT0 helper / push index
   PLT[n]  =  jmp *GOT[n]  (loops into resolver path)

First call:

   CPU hits PLT[n] -> resolver:
       resolves symbol "printf"
       GOT[n] <- &printf in libc
       tail-call to printf

Subsequent calls:

   PLT[n]: jmp *GOT[n]  -> direct jump to libc printf
```

This is why **`LD_BIND_NOW=1`** or `RTLD_NOW` slows startup slightly but avoids surprises during execution and helps **full RELRO** hardening.

### 7.3 Dynamic symbol resolution (simplified)

- Hash / GNU hash maps **name** → **symbol index** in `.dynsym`.
- **Scoped** lookup: the **global symbol table** is a linked structure; load order matters for unversioned symbols (**symbol interposition**).
- **GNU unique** / **protected** / visibility reduce surprise overrides.

### 7.4 `LD_PRELOAD` and symbol interposition

**`LD_PRELOAD`** injects a shared object **before** others in the search order. If it exports the same **global symbol name** as `libc`, calls resolved to that interface may be **intercepted** (used for tracing, mitigations, testing).

**Caveats:**

- Interposing **not** meant to be stable for everything (`malloc` is special-cased in various ways; some symbols use **IFUNC** resolvers).
- Modern hardened programs may use **direct bindings** (`-Wl,-Bdirect`) or **symbol versioning** to reduce accidental interposition.

### 7.5 RELRO — partial vs full

| Mode | Behavior |
|------|----------|
| **Partial RELRO** | GOT **`.got.plt`** remains writable after startup if lazy binding used (PLT targets patched). |
| **Full RELRO** | Resolve everything at load (`BIND_NOW`); mark GOT / sensitive regions read-only via `PT_GNU_RELRO` — mitigates GOT overwrite attacks. |

Check flags (Linux):

- `readelf -l` shows **GNU_RELRO** segment.
- `readelf -d` may show `BIND_NOW`.

---

## 8. ELF Types (`e_type`)

| Value | Name | Role |
|-------|------|------|
| `ET_REL` | Relocatable | `.o` from compiler; not runnable alone. |
| `ET_EXEC` | Executable | Fixed-load (non-PIE classic) executable; entry at absolute-ish layout. |
| `ET_DYN` | Shared object | `.so` **or** **PIE** executable (same file kind; offset `0x0000` is **not** a load address for mapping in general). |
| `ET_CORE` | Core dump | Memory image for post-mortem debug (`gcore`, crash dumps). |

**When each is used**

- **`ET_REL`:** compiler output for linking archives or final link.
- **`ET_EXEC`:** legacy non-PIE Linux exe (less common now on hardened distros).
- **`ET_DYN`:** shared libraries; **default** for modern executables with ASLR (PIE).
- **`ET_CORE`:** debugging after crashes (not executed).

---

## 9. Practical Examples

### 9.1 Using `readelf` (GNU/Linux)

Typical commands:

```bash
readelf -h ./a.out              # ELF header
readelf -l ./a.out              # Program headers (segments)
readelf -S ./a.out              # Section headers
readelf -s ./a.out              # Symbol tables
readelf -r ./a.out              # Relocations
readelf -d ./a.out              # Dynamic section
readelf --dyn-syms ./a.out      # Dynamic symbols only
```

**Representative `readelf -h` output** (PIE executable on x86-64 Linux):

```
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)   # PIE shows as DYN
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1140                     # example only
  Start of program headers:          64 (bytes into file)
  Start of section headers:          14640 (bytes into file)    # example only
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program header:            56 (bytes)
  Number of program headers:         13                       # example
  Size of section header:            64 (bytes)
  Number of section headers:         31                       # example
  Section header string table index: 30
```

**Representative `readelf -S` fragment:**

```
Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [11] .text             PROGBITS        0000000000001060 001060 0001a2 00  AX  0   0 16
  [14] .rodata           PROGBITS        0000000000002000 002000 000016 00   A  0   0  8
  [24] .data             PROGBITS        0000000000003db8 002db8 000010 00  WA  0   0  8
  [25] .bss              NOBITS          0000000000003dc8 002dc8 000008 00  WA  0   0  8
```

### 9.2 Using `objdump`

```bash
objdump -f ./a.out                 # File header summary
objdump -h ./a.out                 # Section summary
objdump -d ./a.out                 # Disassembly (.text)
objdump -d -M intel ./a.out        # Intel syntax (GNU objdump)
objdump -R ./a.out                 # Dynamic relocations
objdump -t ./a.out                 # Symbol table
```

**Representative `objdump -f` fragment (Linux ELF):**

```
./a.out:     file format elf64-x86-64
architecture: i386:x86-64, flags 0x00000150:
HAS_RELOC, EXEC_P, D_PAGED
start address 0x0000000000001140
```

On **macOS**, `objdump` typically reports **`Mach-O`** for system binaries — use Linux or cross-binutils for ELF practice.

### 9.3 Minimal ELF by hand (educational hex sketch)

Writing a **valid, minimal** ELF that an OS will load is platform- and ABI-specific; the following is a **toy illustration** of the **magic + class** bytes only:

```
Offset  Hex bytes                        Meaning
------  -----------------------------    -------
0x00    7f 45 4c 46                     ELF magic
0x04    02                              ELFCLASS64
0x05    01                              ELFDATA2LSB
0x06    01                              EV_CURRENT
```

A famously tiny ELF is the **“Smallest ELF”** exploriums (often ~45–120 bytes) that craft a valid **`ET_EXEC`** with one loadable segment and `_start` sysexit—**fantastic for learning**, but fragile across kernel versions. For real tasks, always let **`ld`** produce binaries or study **manuals** + **kernel `fs/binfmt_elf.c`** expectations.

**Legitimate exercise:** hex-edit **`e_type`**, **`e_entry`**, or **`PT_LOAD`** fields on a **copy** of a known-good binary, then observe `readelf` failures—superb for internalizing header/segment roles.

### 9.4 Parsing ELF programmatically in C

Below: minimal **64-bit LE** header read + sanity checks (Linux `<elf.h>`):

```c
#define _POSIX_C_SOURCE 200809L
#include <elf.h>
#include <errno.h>
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

static void die(const char *msg) {
    perror(msg);
    exit(EXIT_FAILURE);
}

int main(int argc, char **argv) {
    if (argc != 2) {
        fprintf(stderr, "usage: %s <elf-file>\n", argv[0]);
        return EXIT_FAILURE;
    }

    int fd = open(argv[1], O_RDONLY);
    if (fd < 0)
        die("open");

    struct stat st;
    if (fstat(fd, &st) < 0)
        die("fstat");

    size_t sz = (size_t)st.st_size;
    if (sz < sizeof(Elf64_Ehdr)) {
        fprintf(stderr, "too small\n");
        return EXIT_FAILURE;
    }

    void *base = mmap(NULL, sz, PROT_READ, MAP_PRIVATE, fd, 0);
    if (base == MAP_FAILED)
        die("mmap");
    close(fd);

    Elf64_Ehdr *eh = (Elf64_Ehdr *)base;
    if (memcmp(eh->e_ident, ELFMAG, SELFMAG) != 0) {
        fprintf(stderr, "bad ELF magic\n");
        return EXIT_FAILURE;
    }
    if (eh->e_ident[EI_CLASS] != ELFCLASS64) {
        fprintf(stderr, "need ELF64 demo\n");
        return EXIT_FAILURE;
    }
    if (eh->e_ident[EI_DATA] != ELFDATA2LSB) {
        fprintf(stderr, "need little-endian demo\n");
        return EXIT_FAILURE;
    }

    printf("type     = %u\n", eh->e_type);
    printf("machine  = %u\n", eh->e_machine);
    printf("entry    = 0x%lx\n", (unsigned long)eh->e_entry);
    printf("phoff    = 0x%lx  phnum=%u phentsize=%u\n",
           (unsigned long)eh->e_phoff, eh->e_phnum, eh->e_phentsize);
    printf("shoff    = 0x%lx  shnum=%u shentsize=%u shstrndx=%u\n",
           (unsigned long)eh->e_shoff, eh->e_shnum, eh->e_shentsize, eh->e_shstrndx);

    if (eh->e_phoff == 0 || eh->e_phnum == 0) {
        printf("no program headers (likely ET_REL .o)\n");
        munmap(base, sz);
        return EXIT_SUCCESS;
    }

    if (eh->e_phoff + (uint64_t)eh->e_phnum * eh->e_phentsize > sz) {
        fprintf(stderr, "program headers out of range\n");
        return EXIT_FAILURE;
    }

    Elf64_Phdr *ph = (Elf64_Phdr *)((uint8_t *)base + eh->e_phoff);
    for (uint16_t i = 0; i < eh->e_phnum; ++i, ++ph) {
        printf("phdr[%u] type=%u off=0x%lx vaddr=0x%lx filesz=0x%lx memsz=0x%lx flags=0x%x\n",
               i,
               ph->p_type,
               (unsigned long)ph->p_offset,
               (unsigned long)ph->p_vaddr,
               (unsigned long)ph->p_filesz,
               (unsigned long)ph->p_memsz,
               ph->p_flags);
    }

    munmap(base, sz);
    return EXIT_SUCCESS;
}
```

Compile on Linux:

```bash
cc -std=c11 -Wall -Wextra -O2 elfpeek.c -o elfpeek
./elfpeek /bin/ls
```

Extend this by:

- Walking **`Elf64_Shdr`** at `e_shoff` for section names via `.shstrtab`.
- Parsing **symbols** via **`SHT_SYMTAB` / `SHT_DYNSYM`**.
- Applying **relocations** only inside tools; the dynamic linker does that for executables.

---

## Further reading (authoritative sources)

- **System V ABI** processor supplement (e.g. *x86-64 psABI*): relocations, calling conventions, PLT/GOT details.
- **Linux man pages:** `elf(5)`, `vdso(7)`.
- **Toolchain docs:** GNU `ld`, `gold`, `lld` linking behavior; `readelf`, `objdump`.
- **Kernel:** Linux `fs/binfmt_elf.c`, ELF loader path.

---

## Appendix: Quick reference — ELF64 program header layout

```
Elf64_Phdr (56 bytes on most Linux targets):

Offset  Field     Size
------  -----     ----
0x00    p_type    4
0x04    p_flags   4
0x08    p_offset  8
0x10    p_vaddr   8
0x18    p_paddr   8
0x20    p_filesz  8
0x28    p_memsz   8
0x30    p_align   8
```

---

*End of tutorial.*
