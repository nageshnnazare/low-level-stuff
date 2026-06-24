# 1.3 — The Toolchain End to End

This chapter is the spine of the whole guide. We follow a tiny C++ program from
source to a running process, naming every tool, every intermediate file, and
every format. Later parts zoom into each stage; here you get the full picture so
nothing feels disconnected.

---

## 1.3.1 The pipeline, named precisely

```
  hello.cpp
     │
     │  ① PREPROCESS   (cpp / clang -E)        — text → text
     ▼
  hello.ii   (translation unit: macros expanded, #includes inlined)
     │
     │  ② COMPILE      (cc1plus / clang -S)    — text → assembly text
     ▼
  hello.s   (target assembly, mnemonics + directives)
     │
     │  ③ ASSEMBLE     (as / clang -c)         — assembly → machine code
     ▼
  hello.o   (relocatable ELF object: code + data + symbols + relocations)
     │
     │  ④ LINK         (ld / lld via the cc driver) — combine + resolve
     ▼
  a.out / hello   (executable ELF, or a .so shared object)
     │
     │  ⑤ LOAD & RUN   (kernel execve + ld.so dynamic loader)
     ▼
  a live process in virtual memory
```

The crucial thing most people miss: **`gcc`/`g++`/`clang` are *drivers*, not
the compiler.** They orchestrate the four sub-tools above. When you run
`g++ hello.cpp`, it silently runs the preprocessor, the compiler proper, the
assembler, and the linker in sequence, deleting the intermediates.

**Try it ▸** Watch the driver call each tool:

```bash
g++ -v hello.cpp 2>&1 | sed -n '1,40p'
# Look for invocations of cc1plus (the compiler), `as` (assembler),
# and `collect2`/`ld` (the linker).
```

---

## 1.3.2 Stage ① — The preprocessor

Pure text transformation. It:

- Expands `#include` by literally pasting file contents.
- Expands macros (`#define`), evaluates `#if`/`#ifdef`.
- Strips comments, handles `#pragma once`, line markers (`# 1 "file"`).

```
  #include <cstdio>          ──▶   [thousands of lines from headers]
  #define N 3                ──▶   (vanishes; uses replaced)
  int a[N];                  ──▶   int a[3];
```

It knows **nothing** about C++ syntax or semantics. Output is still C++ text.

**Try it ▸** `g++ -E hello.cpp | wc -l` — be shocked at how many lines a single
`#include <iostream>` drags in.

---

## 1.3.3 Stage ② — The compiler proper

This is the big one (and the subject of compiler courses, not this guide). It
turns the translation unit into **target assembly**:

```
  front end:  lex → parse → semantic analysis → build AST/IR
  middle end: optimize IR (SSA passes: inlining, DCE, vectorization, ...)
  back end:   instruction selection → register allocation → emit .s
```

For our purposes, the compiler's relevant *output decisions* are:

- Which **sections** symbols go in (`.text`, `.data`, `.bss`, `.rodata`,
  `.tdata`, custom sections...).
- Symbol **binding & visibility** (global vs local; default/hidden).
- **C++ name mangling** of every symbol (Part 7.3).
- Emitting **relocations** for anything whose final address it can't know.
- Optionally emitting **DWARF** debug info (`-g`) into `.debug_*` sections.

**Try it ▸** Stop after compilation to read the assembly:

```bash
g++ -O2 -S -fverbose-asm hello.cpp -o hello.s
# `-S` = emit .s and stop. `-fverbose-asm` adds helpful comments.
```

---

## 1.3.4 Stage ③ — The assembler

`as` (GNU assembler, part of binutils) converts the textual `.s` into a
**relocatable object file** (`.o`), an ELF `ET_REL` file. It:

- Translates each mnemonic into its binary **encoding**.
- Maintains a **location counter** per section.
- Builds the **symbol table** from labels and `.globl`/`.local` directives.
- Emits **relocation records** for operands referencing symbols whose addresses
  aren't yet known.
- Does **not** resolve cross-file references — that's the linker's job.

```
   .s (text)                         .o (binary ELF ET_REL)
   ─────────                         ─────────────────────
   movl $5, count(%rip)    ──as──▶   machine bytes + a relocation
                                     "patch the disp32 here to point at `count`"
```

We dedicate all of Part 3 to this.

**Try it ▸** `g++ -c hello.cpp -o hello.o && file hello.o`
→ `ELF 64-bit LSB relocatable, x86-64, ...`

---

## 1.3.5 Stage ④ — The linker

The linker (`ld`, or faster modern ones `lld`/`gold`/`mold`) combines one or
more `.o` files and libraries into a single executable or shared object. Its
core jobs:

```
  Inputs: a.o  b.o  libc.a  libfoo.so
                    │
                    ▼
  ┌───────────────────────────────────────────────┐
  │ 1. Collect sections from all inputs           │
  │ 2. Merge same-named sections (.text + .text)  │
  │ 3. Lay out segments & assign virtual addresses│
  │ 4. Resolve symbols (match undef → def)        │
  │ 5. Apply relocations (patch the bytes)        │
  │ 6. Generate dynamic linking metadata          │
  │ 7. Write the output ELF                       │
  └───────────────────────────────────────────────┘
                    │
                    ▼
              a.out (ET_EXEC or ET_DYN)
```

```
  Section merging (simplified):

   a.o            b.o                  a.out
  ┌────────┐     ┌────────┐          ┌──────────────┐
  │ .text  │     │ .text  │   ───▶   │ .text (a+b)  │  ← concatenated & placed
  ├────────┤     ├────────┤          ├──────────────┤
  │ .data  │     │ .data  │   ───▶   │ .data (a+b)  │
  └────────┘     └────────┘          └──────────────┘
```

Symbol resolution is where `undefined reference to 'foo'` and `multiple
definition of 'foo'` come from. Relocation application is where the linker
finally writes real addresses into the holes the assembler left. All of Part 5
is about this.

**Try it ▸** Link with the linker's own map output:

```bash
g++ hello.o -o hello -Wl,-Map=hello.map
# hello.map shows every input section and its final address.
```

---

## 1.3.6 Stage ⑤ — Loading and running

You type `./hello`. The shell calls `fork()` then `execve("./hello", ...)`.
The kernel:

1. Reads the ELF header & **program headers** (not section headers!).
2. `mmap`s the `PT_LOAD` segments into the new address space.
3. Sees a `PT_INTERP` segment naming the **dynamic loader** (e.g.
   `/lib64/ld-linux-x86-64.so.2`) and hands control to *it*, not your `main`.
4. The dynamic loader maps the shared libraries (`libc.so`, `libstdc++.so`,
   ...), applies dynamic relocations, runs init functions, and finally jumps to
   your program's `_start`.
5. `_start` (from C runtime `crt1.o`) sets up `argc`/`argv`/`env`, runs global
   constructors via `.init_array`, then calls `main`.

```
  execve ──▶ kernel maps PT_LOADs ──▶ ld.so ──▶ relocate libs
                                                 run .init_array
                                                 ──▶ _start ──▶ __libc_start_main
                                                                 ──▶ main()
```

This is Part 5.5 in full. The key takeaway now: **a "static-looking" C++
program is, by default, a dynamically-linked thing that the loader assembles at
startup.**

---

## 1.3.7 The two faces of an ELF file

This is the single most important conceptual hinge of the entire guide, so we
introduce it early:

```
   THE SAME BYTES, TWO VIEWS

   LINKING VIEW (what ld reads)        EXECUTION VIEW (what the loader reads)
   ────────────────────────────       ────────────────────────────────────
   Section Header Table               Program Header Table
   describes SECTIONS:                describes SEGMENTS:
     .text  .rodata  .data            PT_LOAD (r-x)  ← .text + .rodata
     .bss   .symtab  .rela.text       PT_LOAD (rw-)  ← .data + .bss
     .debug_info  ...                 PT_INTERP, PT_DYNAMIC, PT_GNU_STACK...
   Fine-grained, many small pieces    Coarse-grained, page-aligned chunks
   Used at LINK time                  Used at LOAD time
```

- **Sections** are the linker's fine-grained units (dozens of them).
- **Segments** are the loader's coarse, page-aligned, permission-homogeneous
  groupings of sections.
- A relocatable `.o` has sections but (usually) no useful program headers. A
  final executable/shared object has both, but the loader only needs the
  program headers; you can strip the section headers and it still runs.

Keep this diagram in your head through Parts 4 and 5.

---

## 1.3.8 Build artifacts cheat sheet

| Extension      | ELF type    | Made by   | Consumed by   | Contents                          |
|----------------|-------------|-----------|---------------|-----------------------------------|
| `.i` / `.ii`   | (text)      | cpp       | compiler      | preprocessed source               |
| `.s` / `.asm`  | (text)      | compiler  | assembler     | assembly mnemonics + directives   |
| `.o` / `.obj`  | `ET_REL`    | assembler | linker        | code/data + symbols + relocations |
| `.a` / `.lib`  | archive     | `ar`      | linker        | bundle of `.o` files (static lib) |
| `.so` / `.dylib`/`.dll` | `ET_DYN` | linker | loader/linker | shared library                 |
| `a.out`/exe    | `ET_EXEC` or `ET_DYN` (PIE) | linker | kernel+loader | the program          |

---

## 1.3.9 Inspecting every stage — your core tools

You'll use these constantly. Bookmark this table; Part 7.5 goes deep.

| Tool         | What it shows                                             |
|--------------|-----------------------------------------------------------|
| `file`       | Quick ELF type / class / arch summary                     |
| `readelf -a` | The canonical, complete ELF structure dump                |
| `objdump -d` | Disassembly with symbols                                  |
| `objdump -r` | Relocation records                                        |
| `nm`         | Symbol table (defined/undefined, types)                   |
| `nm -C`      | …with C++ names demangled                                 |
| `c++filt`    | Demangle a single mangled name                            |
| `ldd`        | Shared library dependencies of an executable              |
| `strings`    | Printable strings in a binary                             |
| `size`       | Section sizes (text/data/bss)                             |
| `addr2line`  | Map an address back to file:line (uses DWARF)             |
| `dwarfdump` / `readelf --debug-dump` | DWARF debug info                  |
| `strace`     | System calls (watch `execve`, `mmap`, `openat` of libs)   |
| `ltrace`     | Library calls (watch PLT resolution)                      |
| `gdb`        | Everything, interactively                                 |

**Try it ▸** The end-to-end demo. Run this and skim each output — you'll
understand all of it by the end of the guide:

```bash
cat > hello.cpp <<'EOF'
#include <cstdio>
int answer = 42;
int main() { std::printf("answer = %d\n", answer); return 0; }
EOF

g++ -g -O0 hello.cpp -o hello
file hello                 # ET_DYN (PIE) on modern distros
readelf -h hello           # the ELF header
readelf -l hello           # program headers (segments) — execution view
readelf -S hello           # section headers — linking view
nm -C hello | grep main    # where is main?
ldd hello                  # which shared libs?
objdump -d hello | sed -n '/<main>:/,/ret/p'   # disassemble main
./hello
```

---

## Summary

- The compiler driver (`g++`/`clang`) orchestrates four tools: preprocessor,
  compiler, assembler, linker — then the OS loader runs the result.
- Each stage has a precise input/output format; this guide zooms into each.
- The assembler produces relocatable objects with *holes* (relocations); the
  linker fills them and lays out memory; the loader maps and finalizes at
  startup.
- An ELF file has **two views**: sections (linking) and segments (execution).
  This duality recurs throughout.
- A small toolbox (`readelf`, `objdump`, `nm`, `gdb`, …) lets you observe every
  stage directly.

You now have the full map. → Part 2 dives into the assembly language itself:
[2.1 — x86-64 architecture for the toolchain hacker](../02-assembly/01-x86-64-architecture.md)
