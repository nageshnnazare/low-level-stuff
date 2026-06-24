# Binary Utilities (Bintools / Binutils)

This tutorial surveys the **GNU Binutils** suite and closely related tools used to inspect, transform, and link object files and executables on Unix-like systems. The primary target format on Linux and many other environments is **ELF** (Executable and Linkable Format).

---

## 1. What Are Bintools?

### 1.1 Terminology

- **Binutils** (GNU Binary Utilities) is the canonical name of the GNU project that implements `as`, `ld`, `objdump`, `readelf`, `nm`, `objcopy`, `strip`, `ar`, `strings`, `size`, `addr2line`, `c++filt`, and related programs.
- People sometimes say **“bintools”** informally to mean “the binary toolchain utilities” (often but not always GNU). In embedded and cross-compilation contexts, the prefix matters: `arm-none-eabi-objdump`, `x86_64-linux-gnu-nm`, etc.

### 1.2 The GNU Binutils Suite and Alternatives

**GNU Binutils** is the traditional companion to **GCC**. It provides:

| Category | Typical tools |
|----------|----------------|
| Assembly | `as` (GNU assembler, often invoked as `gcc -c` on `.s` files) |
| Linking | `ld` (often wrapped by `gcc` or `clang`) |
| Inspection | `objdump`, `readelf`, `nm`, `strings`, `size` |
| Transformation | `objcopy`, `strip` |
| Archives | `ar`, `ranlib` |
| Debugging aids | `addr2line`, `c++filt` |

**Alternatives and complements:**

- **LLVM** provides parallel tools: `llvm-objdump`, `llvm-readelf`, `llvm-nm`, `llvm-objcopy`, `llvm-strip`, `llvm-ar`, `llvm-symbolizer`, often with richer diagnostics and consistent LLVM IR-related workflows.
- **Platform-specific** loaders and formats: **Mach-O** on macOS (inspection often via `otool`, `nm`, `dyld` APIs); **PE** on Windows (`dumpbin`, `link.exe`). Many concepts (sections, symbols, relocations) echo ELF.
- **Vendor SDKs** ship cross-`objcopy`/`readelf` prefixed for the target triplet.

### 1.3 Toolchain Pipeline (Where Binutils Fit)

From source to running program, the usual pipeline is:

```
  .c / .cpp / .S                preprocessed          assembly
       |                              |                    |
       v                              v                    v
  +---------+                +---------------+      +------------+
  | Preproc |  cc1/cc1plus   | Compiler      |  as  | Assembler  |
  | (cpp)   | -------------->| (GCC/Clang    |----->| (gas)      |
  +---------+                |  backend)     |      +------------+
                             +---------------+             |
                                    |                      |
                                    |                      v
                                    |              +---------------+
                                    +------------->| Relocatable   |
                                                   | object (.o)   |
                                                   +---------------+
                                                          |
                                                          | ld (+ crt, libs)
                                                          v
                                                   +---------------+
                                                   | Executable /  |
                                                   | shared lib    |
                                                   +---------------+
```

**Role of each stage:**

1. **Preprocessor** expands `#include`, `#define`, and conditional compilation. Output is a **translation unit** fed to the compiler proper.
2. **Compiler** emits **assembly** (conceptually); on disk you may see `.s` or it may stay internal.
3. **Assembler (`as`)** turns assembly into **relocatable object files** (`.o`), containing **sections**, **symbols**, and **relocation records**.
4. **Linker (`ld`)** merges objects and libraries, resolves symbols, applies relocations, and may perform layout (segments, GOT/PLT for dynamic linking). Output is an **executable** or **shared object**.

Binutils tools mostly operate on **object files and linked images**, not on high-level source (except when interleaving source with `-S` in `objdump`, which needs debug info).

---

## 2. The Assembler (`as` / GAS)

### 2.1 How It Works

**GNU Assembler (GAS)** reads assembly and produces an object file:

- Maps **mnemonics** to **instruction encodings** for the target ISA.
- Organizes bytes into **sections** (e.g. `.text`, `.data`, `.bss`).
- Emits **symbols** (labels, `.globl` names) and **relocations** when the final value is unknown until link time (e.g. calls to external functions, use of symbols defined in other `.o` files).
- Respects **directives** that control alignment, data emission, symbol attributes, and metadata.

**Invocation examples:**

```bash
# Assemble to object file (explicit)
as -o hello.o hello.s

# Usually you use the compiler to drive GAS:
gcc -c hello.s -o hello.o
```

### 2.2 AT&T Syntax Specificities (GNU `as` on x86)

On x86, GAS defaults to **AT&T syntax** (not Intel unless configured). Distinctive rules:

| Feature | AT&T (GAS) typical form |
|---------|-------------------------|
| Operand order | `instr source, dest` (opposite of Intel `dest, source`) |
| Register names | Prefixed with `%`: `%eax`, `%rdi` |
| Immediate values | Prefixed with `$`: `$42`, `$symbol` |
| Memory | `disp(base,index,scale)`: `8(%rsp)`, `(%rax,%rbx,4)` |
| Instruction size suffix | `b`/`w`/`l`/`q` on many mnemonics: `movl`, `movq` |

Example (x86-64):

```asm
        .text
        .globl  _start
_start:
        movq    $1, %rax          # syscall number (exit) — illustrative
        movq    $0, %rdi          # status
        syscall
```

*Note: Real “Hello, world” or Linux syscall sequences differ; this illustrates syntax only.*

### 2.3 Directives (Overview)

Directives begin with `.` (dot). Common groups:

**Sections and linkage**

| Directive | Meaning |
|-----------|---------|
| `.text` | Code section (executable, often read-only in final binary). |
| `.data` | Initialized writable data. |
| `.bss` | Uninitialized writable data (zeroed by loader). |
| `.section name, "flags"` | Select or create a custom section. Flags vary by target; ELF uses string like `"aw"`, `"ax"`, etc. |
| `.global sym` / `.globl sym` | Export symbol to linker (visible outside this file). |
| `.local sym` | Symbol not exported outside the object (ELF local binding). |
| `.weak sym` | Weak symbol (overridable by a strong definition elsewhere). |
| `.type sym, @function` / `@object` | ELF-specific symbol type metadata. |
| `.size sym, expr` | ELF: size of symbol (often `.-sym`). |
| `.align n` | Align location counter (interpretation of `n` is **target-dependent**—bytes or power-of-two). |

**Data emission**

| Directive | Meaning |
|-----------|---------|
| `.byte` / `.short` / `.word` / `.long` / `.quad` | Emit integers of given width. |
| `.ascii "str"` | String without NUL terminator. |
| `.asciz "str"` / `.string "str"` | NUL-terminated string (GAS: `.asciz` / `.string` usage can vary slightly by arch—consult the manual). |
| `.space n [,fill]` | Reserve `n` bytes, optional fill byte. |
| `.equ name, expr` | Define an absolute symbol. |

**Debug / file tracking**

| Directive | Meaning |
|-----------|---------|
| `.file` / `.loc` | Debug line info when compiling with `-g` from C often uses these internally. |

### 2.4 Practical GAS Example

Minimal two-file example illustrating a global symbol and external reference:

**`main.s`:**

```asm
        .text
        .globl  main
        .type   main, @function
main:
        call    helper           # relocation: R_X86_64_PLT32 or similar
        xorl    %eax, %eax
        ret
        .size   main, .-main
```

**`helper.s`:**

```asm
        .text
        .globl  helper
        .type   helper, @function
helper:
        ret
        .size   helper, .-helper
```

**Build:**

```bash
as -o main.o main.s
as -o helper.o helper.s
ld -o prog main.o helper.o    # simplistic; real linking often needs crt and libc
```

After `gcc` linking, `objdump -d prog` shows the combined disassembly.

---

## 3. The Linker (`ld`)

### 3.1 Overview

The **linker** assigns **final addresses** to sections, merges input objects, resolves **undefined symbols** against archives and shared libraries, applies **relocations**, and writes the output ELF layout (including **program headers** for executables/DSOs).

GNU `ld` supports complex **linker scripts** (MEMORY, SECTIONS), **LTO** integration when driven by the compiler, and many options for security hardening (e.g. `-z relro`, `-z now`).

> **Note:** For ELF internals, relocation types, PLT/GOT, and script details, see the companion document **`06_linker_internals.md`** (when available in this series).

### 3.2 Basic Usage

```bash
# Typical user never calls ld alone; gcc/clang provides crt files and default libs:
gcc main.c -o main

# Explicit static link (illustrative; platform-dependent):
gcc -static main.c -o main_static
```

**Minimal `ld` example (often fails without startup files):**

```bash
ld -o prog main.o -lc          # needs correct crt and library paths on your system
```

In practice, use `gcc -v` to see the full `collect2` / `ld` invocation your compiler uses.

### 3.3 Common `ld` Flags (GNU)

| Flag | Purpose |
|------|---------|
| `-o file` | Output file name. |
| `-e sym` | Entry symbol (default often `_start` for bare metal; `main` for hosted via crt). |
| `-T script.ld` | Use linker script. |
| `-L dir` | Library search path. |
| `-lname` | Link `libname.so` / `libname.a`. |
| `-static` | Prefer static archives where possible. |
| `-shared` | Build shared object. |
| `-r` | Relocatable output (partial link). |
| `--gc-sections` | Drop unreferenced sections (often with `-ffunction-sections -fdata-sections` in compiler). |
| `-Map=file` | Write link map (symbol addresses, memory regions). |
| `-z relro` / `-z now` | Hardening: full RELRO and immediate binding (overview in §15). |

---

## 4. `objdump` — Disassembler and Object Inspector

`objdump` is versatile: disassembly, headers, symbols, relocations, DWARF metadata (depending on options and build).

### 4.1 Disassembling Code

```bash
objdump -d prog              # Disassemble executable sections only
objdump -D prog              # Disassemble all sections (can include data as “instructions”)
objdump -d -M intel prog     # Intel syntax on x86 (GNU objdump)
objdump -d -S prog           # Interleave source when debug info present
```

**Typical snippet of `-d` output (illustrative):**

```
0000000000401000 <main>:
  401000:	55                   	push   %rbp
  401001:	48 89 e5             	mov    %rsp,%rbp
  ...
```

**`-S` interleaving** requires compile with debug (`-g`). You may see:

```
/path/to/file.c:10
  401012:	e8 00 00 00 00       	call   ...
```

### 4.2 Examining Sections (`-h`, `-j`)

```bash
objdump -h prog.o            # Section headers (VMA, size, flags)
objdump -h -j .text prog.o   # Restrict to one section
```

Example-style output fragment:

```
Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000001e  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
```

### 4.3 Symbols (`-t`, `-T`)

```bash
objdump -t prog.o            # Symbol table (all)
objdump -T prog              # Dynamic symbol table (DSOs/executables with dyn syms)
```

### 4.4 Relocations (`-r`, `-R`)

```bash
objdump -r prog.o            # Relocation entries for relocatable file
objdump -R prog              # Dynamic relocations (if present)
```

You may see entries like `R_X86_64_PC32`, `R_X86_64_PLT32`, `R_X86_64_REX_GOTPCRELX`, etc.

### 4.5 File Headers (`-f`)

```bash
objdump -f prog
```

Typical lines include architecture, `start address`, and `format elf64-x86-64`.

### 4.6 Other Useful Flags

| Flag | Role |
|------|------|
| `-x` | All headers (comprehensive dump) |
| `-p` | Private headers (e.g. program headers summary on some versions) |
| `-s` | Full section hex dump |
| `-C` | Demangle C++ symbols |
| `-g` | Display debug info (format-dependent) |

### 4.7 Worked Example: Inspecting a Relocatable Object

Suppose you built `hello.o` from `hello.c` with `gcc -c hello.c`. A typical exploration sequence:

```bash
objdump -f hello.o
objdump -h hello.o
objdump -t hello.o
objdump -r hello.o
objdump -d hello.o
```

**`objdump -f hello.o`** — identifies the file format and architecture (values depend on your platform):

```
hello.o:     file format elf64-x86-64
architecture: i386:x86-64, flags 0x00000011:
HAS_RELOC, HAS_SYMS
start address 0x0000000000000000
```

**`objdump -r hello.o`** — relocation records that the linker will patch (types and offsets vary with code):

```
RELOCATION RECORDS FOR [.text]:
OFFSET           TYPE              VALUE
0000000000000007 R_X86_64_PC32     puts-0x00000004
000000000000000e R_X86_64_PC32     __stack_chk_fail-0x00000004
```

This tells you: at offset `0x7` into `.text`, the link editor must apply a PC-relative fixup for `puts`, etc.

**`objdump -d hello.o`** — disassembly often shows placeholders (`0` displacement) where relocations apply:

```
0000000000000000 <main>:
   0:	55                   	push   %rbp
   ...
   7:	e8 00 00 00 00       	call   8 <main+0x8>    # R_X86_64_PC32 to puts
```

After linking, re-run `objdump -d` on `a.out` and the `call` target will be concrete.

---

## 5. `readelf` — ELF File Analyzer

`readelf` reads ELF structures **without** relying on BFD the way `objdump` does for some paths; it is often the most direct ELF-only view.

### 5.1 ELF File Header (`-h`)

```bash
readelf -h prog
```

Shows `Class`, `Data`, `Version`, `OS/ABI`, `Type` (EXEC, DYN, REL), `Machine`, `Entry point address`, `Start of program headers`, `Start of section headers`, `Flags`, etc.

### 5.2 Section Headers (`-S`)

```bash
readelf -S prog
```

Lists `.text`, `.rodata`, `.data`, `.bss`, `.symtab`, `.strtab`, `.dynsym`, `.rela.dyn`, `.rela.plt`, debug sections.

### 5.3 Program Headers (`-l`) — Segments

```bash
readelf -l prog
```

Shows **LOAD**, **INTERP** (`/lib64/ld-linux-x86-64.so.2`), **DYNAMIC**, **GNU_STACK**, **GNU_RELRO**, etc.—critical for security attributes (e.g. whether RELRO is partial or full).

### 5.4 Symbol Table (`-s`)

```bash
readelf -s prog
```

Columns include `Num`, `Value`, `Size`, `Type`, `Bind`, `Vis`, `Ndx` (section index or `ABS`, `UND`).

### 5.5 Dynamic Section (`-d`)

```bash
readelf -d prog.so
```

Shows `NEEDED`, `SONAME`, `RUNPATH`/`RPATH`, `INIT`, `FINI`, `FLAGS`, `FLAGS_1`, `BIND_NOW`, etc.

### 5.6 Relocations (`-r`)

```bash
readelf -r prog.o
readelf -r prog            # dynamic relocations on linked file
```

### 5.7 Notes (`-n`)

```bash
readelf -n prog
```

May show **GNU ABI** tag, **build-id** (`GNU_BUILD_ID`), **version** notes from the linker.

### 5.8 DWARF (`--debug-dump`)

```bash
readelf --debug-dump=info prog      # .debug_info (huge)
readelf --debug-dump=abbrev prog
readelf --debug-dump=line prog
readelf --debug-dump=frames prog
```

Use selectively; full `info` dump is verbose. Often `llvm-dwarfdump` is nicer for DWARF browsing when available.

### 5.9 Comprehensive Example Session (Pattern)

```bash
readelf -hW -S -s prog        # File header + section headers + symbol table (-W = wide columns)
readelf -l prog | grep RELRO
readelf -d ./libfoo.so | egrep 'NEEDED|RUNPATH|RPATH'
```

*Note: Option letters are case-sensitive: `-S` = section headers, `-s` = symbol table.*

### 5.10 Worked Example: ELF Header and Program Headers

**`readelf -h /bin/true`** (GNU/Linux x86-64, illustrative; addresses may differ):

```
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x....
  Start of program headers:          (bytes into file)
  Start of section headers:          (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         NN
  Size of section headers:          64 (bytes)
  Number of section headers:        MM
  Section header string table index: K
```

`Type: DYN` with `file` reporting “dynamically linked” is typical for **PIE** executables on modern distros.

**`readelf -l /bin/true`** — segment view (excerpt pattern):

```
Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           ...      ...                ...                ...      ...      R   ...
  INTERP         ...      ...                ...                ...      ...      R   ...
  LOAD           ...      ...                ...                ...      ...   R E  ...
  LOAD           ...      ...                ...                ...      ...   RW   ...
  DYNAMIC        ...      ...                ...                ...      ...   RW   ...
  NOTE           ...      ...                ...                ...      ...   R    ...
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW   16
  GNU_RELRO      ...      ...                ...                ...      ...    R    ...
```

**Reading flags:**

- **`GNU_STACK` with `R` only (no `E`)** → non-executable stack (NX).
- **`GNU_RELRO`** present → a **read-only after relocation** region exists (RELRO); “full” vs “partial” depends on how much of the GOT/related areas are covered—compare segment size with dynamic tags and linker flags.

**`readelf -d /bin/true`** — dynamic tags (pattern):

```
Dynamic section at offset 0x.... contains NN entries:
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
 0x0000001e (FLAGS)                      BIND_NOW
 ...
```

`BIND_NOW` (often combined with full RELRO) implies **immediate symbol resolution** at load time—PLT/GOT hardening pattern.

---

## 6. `nm` — Symbol Table Viewer

### 6.1 Symbol Type Letters (common ELF/GNU `nm`)

Single-letter **type** in the middle column (exact layout depends on format):

| Char | Typical meaning |
|------|-----------------|
| `A` | Absolute symbol (not relocatable) |
| `B` / `b` | BSS (uninitialized data); global / local |
| `C` | Common symbol (legacy “common block”) |
| `D` / `d` | Initialized data; global / local |
| `G` / `g` | Initialized data (small model / GOT-related on some targets) |
| `N` | Debug symbol |
| `R` / `r` | Read-only data |
| `S` / `s` | Small data (target-specific) |
| `T` / `t` | Text (code); global / local |
| `U` | Undefined (referenced but not defined in this file) |
| `V` / `v` | Weak object |
| `W` / `w` | Weak symbol (not classified as weak object by this letter on all systems—verify with `man nm` on your OS) |
| `?` | Unknown |

**Interpretation tip:** Case often distinguishes **global** (uppercase) vs **local** (lowercase).

### 6.2 Sorting and Filtering

```bash
nm prog.o
nm -n prog                  # Sort numerically by address
nm -S prog                  # Show symbol sizes
nm -g prog                  # Extern-only (defined symbols)
nm -u prog.o                # Undefined only
nm --defined-only prog
nm --print-size --size-sort prog | tail
```

### 6.3 Demangling (`-C`)

```bash
nm -C libstdc++.so | less
```

Turns Itanium-mangled names into human-readable C++ signatures.

### 6.4 Debugging Use Cases

- **Unresolved link errors:** `nm -u foo.o` lists what `foo.o` needs from other objects.
- **Who defines `symbol`?** `nm -A *.o | grep ' symbol$'` or with `c++filt`.
- **Size creep:** `nm --print-size --size-sort --radix=d` on final binary (static symbols may be stripped).

---

## 7. `objcopy` — Object File Transformer

### 7.1 Stripping Debug Info (Copy-Time)

```bash
objcopy --only-keep-debug prog prog.debug
objcopy --strip-debug prog prog.nodebug
# Or strip fully with strip/objcopy --strip-all
```

Workflow: keep `prog.debug` for `gdb` / `eu-unstrip` style workflows while shipping small binaries.

### 7.2 Converting Formats

```bash
objcopy -O binary prog prog.bin           # Raw memory image (care: segments overlap nuances)
objcopy -O ihex   prog prog.hex
objcopy -O srec   prog prog.srec
```

Embedded flows often require a **flash offset**; you may need `--gap-fill` or linker script `LMA` discipline.

### 7.3 Adding / Removing Sections

```bash
objcopy --add-section .metadata=meta.txt --set-section-flags .metadata=readonly prog prog2
objcopy --remove-section=.comment prog prog2
```

### 7.4 Changing Section Flags

```bash
objcopy --rename-section .old=.new,alloc,load,readonly,code,contents file.o out.o
```

Exact flag spelling varies; consult `objcopy --help` and the manual.

### 7.5 Extracting Sections

```bash
objcopy -j .data -O binary prog data.bin
```

---

## 8. `strip` — Symbol Stripper

### 8.1 What Stripping Does

`strip` removes (or makes non-alloc) symbol table and debug sections from ELF files to:

- **Reduce file size**
- **Obfuscate** symbol names (weak obfuscation only)
- **Avoid leaking** internal function names in production (still reversible with enough analysis)

It does **not** remove machine code needed to run the program (unless you removed sections incorrectly).

### 8.2 Strip Levels (GNU)

Common options:

| Option | Effect (typical) |
|--------|-------------------|
| `--strip-debug` | Remove debug sections only |
| `--strip-unneeded` | Remove symbols not needed for relocation / dynamic linking |
| `--strip-all` | Strip all symbols and relocation info **when safe** (not for objects you still link) |
| `--strip-symbol=name` | Targeted removal |

**Warning:** Stripping **relocatable `.o` files** aggressively can break linking. Strip **final executables** or use `objcopy --only-keep-debug` splits.

### 8.3 Keeping Specific Symbols

Use linker version scripts for visibility, or:

```bash
strip -N unwanted_sym prog
```

For fine control at link time, **version scripts** and **`-Wl,--export-dynamic`** patterns are preferred.

### 8.4 Impact on Debugging and Size

- **Debugging:** Without DWARF (`.debug_*`) or separate debug file, backtraces degrade (`addr2line` fails; gdb has less source-level insight).
- **Size:** Debug info can dominate binary size; production builds often use **split debug** (`-g` + `objcopy --only-keep-debug` + `strip`).

---

## 9. `ar` — Archive Manager

### 9.1 Creating Static Libraries

```bash
gcc -c foo.c bar.c
ar rcs libfoobar.a foo.o bar.o
```

`rcs`: **replace/create** members, **write symbol index** (see `ranlib`).

### 9.2 Operations

| Op | Meaning |
|----|---------|
| `r` | Insert/replace `.o` members |
| `t` | List table of contents |
| `x` | Extract members |
| `d` | Delete members |
| `p` | Print member to stdout |
| `q` | Quick append (no symbol table update until `ranlib`) |

Examples:

```bash
ar t libfoobar.a
ar x libfoobar.a foo.o
```

### 9.3 Thin Archives

```bash
ar rcsT libthin.a a.o b.o
```

Thin archive: stores **paths** to `.o` files instead of embedding full contents—fast for large builds, fragile if `.o` move.

### 9.4 `ranlib` and the Archive Index

Historically `ranlib` wrote a **symbol index** (`__.SYMDEF` on older systems). Modern `ar s` or `ar rcs` embeds the index so the linker can quickly find which member defines `symbol`.

```bash
ranlib libfoobar.a        # Regenerate index if needed
```

---

## 10. `strings` — String Finder

### 10.1 Basic Usage

```bash
strings /bin/ls | head
strings -a binary         # Scan whole file (not only data sections)
```

Default: finds NUL-terminated or length-prefixed printable sequences in **initialized loaded sections**; `-a` scans **all** bytes (slower, noisier, common in forensics).

### 10.2 Minimum Length

```bash
strings -n 8 binary       # Min length 8 (default often 4)
```

### 10.3 Encoding / Format Options (GNU)

Depending on version:

```bash
strings -e s binary       # 16-bit little-endian (UTF-16-ish heuristics)
strings -e l              # 32-bit little-endian
strings -tx binary        # Print file offset in hex
```

Consult `man strings` on your distro for supported `-e` letters.

### 10.4 Forensics / Reverse Engineering

- **Identify libraries**, URLs, error messages, file paths.
- **Leakage**: passwords in binaries (should not happen); keys (should not happen).
- **Pair with** `objdump -s`, `readelf -x .rodata`, `xxd`.

Example pattern:

```bash
strings -tx malware.bin | grep -i 'http'
```

---

## 11. `size` — Section Size Reporter

### 11.1 Text, Data, BSS

```bash
size prog
```

Typical columns:

```
   text    data     bss     dec     hex filename
   xxxx     xxx     xxx    xxxxx    xxxx prog
```

- **text**: code + often **read-only** constants (`.rodata`) depending on linker accounting.
- **data**: initialized writable.
- **bss**: zero-initialized writable (occupies **VM** space but often not file storage).

### 11.2 Berkeley vs SysV

```bash
size -Berkeley prog
size -SysV prog
```

**SysV** may list per-section breakdown; **Berkeley** classic summary. Useful when comparing embedded map files vs `size` totals.

---

## 12. `addr2line` — Address to Source Mapper

### 12.1 Basic Mapping

Compile with `-g` (and avoid stripping debug, or use split debug):

```bash
addr2line -e prog 0x401030
addr2line -e prog -a -f 0x401030 0x401040
```

Output resembles:

```
parse_input
/path/src.c:42
```

### 12.2 Crash Addresses

From a crash log / backtrace:

1. Ensure **PIE** vs **fixed** load address: for PIE, offsets are often **relative to mapped base**; you may need `base + offset`.
2. Use **unstripped** binary or debug file with matching **build-id**.

```bash
# If you have ASLR base from /proc/pid/maps, add appropriately.
addr2line -e prog -fCi 0x555555554000
```

### 12.3 Requirements

- Needs **DWARF** (or at least line tables).
- For inlined code, prefer `llvm-symbolizer` or `gdb` with proper debug.

---

## 13. `c++filt` — Name Demangler

### 13.1 Why Name Mangling Exists

C++ allows **function overloading** and **namespaces**; the linker sees flat symbol names. Compilers encode **type information** into the symbol string (**mangling**) so `void foo(int)` and `void foo(float)` do not collide.

### 13.2 Itanium ABI (GCC/Clang on most Unix)

Example mangled:

```
_ZN3Foo3barEi
```

Demangle:

```bash
echo '_ZN3Foo3barEi' | c++filt
# Foo::bar(int)
```

### 13.3 Practical Examples

```bash
nm -C my.o                              # nm demangles
objdump -C -t my.o
c++filt < mangled_syms.txt
```

For **Rust** or other languages, use their demanglers (`rustfilt`, etc.)—`c++filt` is C++-centric.

---

## 14. LLVM Alternatives

| GNU | LLVM counterpart | Notes |
|-----|------------------|-------|
| `objdump` | `llvm-objdump` | Often similar flags; some output formatting differs; good for LLVM-centric toolchains. |
| `readelf` | `llvm-readelf` | ELF-focused like GNU `readelf`. |
| `nm` | `llvm-nm` | Symbol listings; demangling with `-C`. |
| `objcopy` | `llvm-objcopy` | Broad feature parity for many workflows; verify edge cases for your target. |
| `strip` | `llvm-strip` | Faster in some pipelines; behavior should be validated in release signing workflows. |
| `ar` | `llvm-ar` | Archive manager for compiler-rt builtins in Clang builds. |
| `addr2line` | `llvm-symbolizer` | Parses addresses with smart DWARF parsing; used by sanitizers. |

**Differences (themes):**

- LLVM tools share libraries (`libObject`, `libDebugInfo`) → **consistent** parsing, sometimes **richer error messages**.
- **Flag compatibility** is high but not 100%; automated scripts should check `--help`.
- For **LTO** bitcode, the LLVM toolchain is authoritative.

---

## 15. Practical Workflows

### 15.1 Debugging a Crash with Bintools

```
Backtrace addr  ->  load matching binary + debug
         |
         v
    readelf -n   (build-id match)
         |
         v
    addr2line / llvm-symbolizer
         |
         v
    objdump -d -S around faulting PC
```

Commands:

```bash
readelf -n ./prog | grep 'Build ID'
eu-unstrip -n --list   # if using elfutils
addr2line -e prog.debug -fi 0xdeadbeef
objdump -d --start-address=0x... --stop-address=0x... prog
```

### 15.2 Analyzing a Binary Without Source

```bash
file prog
readelf -hls prog
strings -a prog | less
objdump -d prog | less
readelf -r prog
ldd prog                    # dynamic deps (if ELF executable/DSO on glibc)
```

Map **entry**, **PLT** stubs, **interesting strings**, **dynamic `NEEDED`**.

### 15.3 Size Optimization Workflow

```bash
# Compile for sections + gc
gcc -ffunction-sections -fdata-sections -Os -c *.c
gcc -Wl,--gc-sections -Wl,-Map=prog.map -o prog *.o
size -A prog
# Identify big symbols:
nm --print-size --size-sort prog | tail -n 20
```

Consider **LTO**, **`-Os` vs `-O2`**, compressing **debug** into split files.

### 15.4 Security Analysis (Quick Checks)

```bash
readelf -l prog | grep -E 'GNU_STACK|GNU_RELRO'
readelf -d prog | grep -E 'BIND_NOW|FLAGS_1'
```

Conceptual checklist:

```
Feature        | What to look for
---------------+-------------------------------------------------
PIE            | `readelf -h`: Type `DYN` + ET_DYN for final exe often; or use `file`/`checksec`
NX / stack exec| `GNU_STACK` RWE vs RW-only (non-executable stack)
RELRO          | `GNU_RELRO` program header; full RELRO + `-z now` ⇒ GOT read-only after relocs
Canary         | calls `__stack_chk_fail` (objdump/strings); compiler flag `-fstack-protector*`
ASLR           | OS + PIE binary (policy-level)
```

Tools like **`checksec`** (where available) automate some of this; understanding `readelf -l` / `-d` is the portable foundation.

---

## Quick Reference Command Matrix

```
Goal                  | Commands
----------------------+---------------------------------------------
Disassemble           | objdump -d [-M intel] [-S] [-C]
ELF layout            | readelf -hlS
Symbols               | nm [-C] [-n] [--print-size]; readelf -s
Relocations           | objdump -r; readelf -r
Dynamic linking       | readelf -d; ldd
Strip / debug split   | strip, objcopy --only-keep-debug
Raw ROM image         | objcopy -O binary
Static library        | ar rcs lib.a *.o
Strings RE            | strings -a -tx
Address to line       | addr2line / llvm-symbolize
Demangle C++          | c++filt; nm -C
```

---

## Further Reading

- GNU Binutils manuals: `info bfd`, `info ld`, `info as`
- ELF: **`03_object_file_formats_elf.md`** in this series (if present)
- Linker deep dive: **`06_linker_internals.md`**

---

*Document version: initial. Examples are representative; always verify flags with `man` / `--help` on your target OS and Binutils version.*
