# The Assembler, Linker & Binary Formats Mastery Guide

> A single, deep, diagram-driven reference for how source code becomes a running
> program: the assembler, the linker, the loader, and the on-disk formats
> (ELF, DWARF, and friends) that glue everything together.
>
> Written for C and **C++ engineers who want expert-level mechanical detail** —
> not hand-waving. Every concept is grounded in bytes, ASCII diagrams, and
> reproducible command-line experiments.

---

## 🚀 New to this? Start here first

If you're **new to assemblers, linkers, and binary formats**, the main chapters
below will feel dense. Begin instead with the gentle, analogy-driven
**[Start Here track (Part 0)](00-start-here/01-read-me-first.md)** — it explains
everything in plain English with everyday analogies (think "publishing a book")
and no prerequisites, *then* hands you off to the detailed parts.

```
   Beginner path:   Part 0 (plain English)  →  Part 1 (foundations)  →  rest of guide
   Already comfy?:  jump straight to Part 4 (ELF) and Part 5 (Linking)
```

The Start Here track:

1. [Read me first](00-start-here/01-read-me-first.md) — orientation, zero prerequisites
2. [The big picture in plain English](00-start-here/02-the-big-picture-plain-english.md) — the whole journey by analogy
3. [What is assembly & the assembler?](00-start-here/03-what-is-assembly-and-the-assembler.md)
4. [What is linking?](00-start-here/04-what-is-linking.md) — the heart, extra-gentle
5. [What are ELF and DWARF?](00-start-here/05-what-are-elf-and-dwarf.md)
6. [Beginner glossary, FAQ & first hands-on](00-start-here/06-beginner-glossary-and-faq.md)

---

## Who this is for

You already write code. You can read a stack trace. But you want to *truly*
understand:

- What an object file actually contains, byte by byte.
- Why a link error says `undefined reference to ...` and how resolution works.
- What the PLT/GOT are and why your shared library calls go through them.
- How a debugger maps a crashing address back to a line of C++.
- How exceptions unwind the stack across optimized frames.
- Why `inline`, `static`, templates, COMDAT, and `-fvisibility` behave the way
  they do at the binary level.

If you finish this guide, you will be able to read `readelf`, `objdump`,
`nm`, `dwarfdump`, and a raw hex dump fluently, and reason about every stage
of the toolchain.

---

## The 30,000-foot map

```
   .c / .cpp                                            a running process
   source                                               in virtual memory
      │                                                        ▲
      │  preprocessor (cpp)                                    │
      ▼                                                        │
   .i / .ii  translation unit                                  │ execve(2)
      │                                                        │ + ld.so
      │  compiler front+middle+back end (cc1 / cc1plus)        │
      ▼                                                        │
   .s  assembly  ◀── human-readable mnemonics                  │
      │                                                        │
      │  assembler (as)                                        │
      ▼                                                        │
   .o  RELOCATABLE object file (ELF ET_REL)                    │
      │                                                        │
      │  linker (ld / lld / gold / mold)                       │
      ▼                                                        │
   a.out / .so  EXECUTABLE (ET_EXEC/ET_DYN) or shared object   │
      │                                                        │
      └────────────────────────────────────────────────────────┘
```

Each arrow is a *tool*. Each box is a *format*. This guide walks every arrow
and dissects every box.

---

## How to read this guide

The chapters are ordered as a **learning path** from fundamentals to advanced
internals. If you are already comfortable with assembly, you can jump straight
to Part 4 (ELF) and Part 5 (Linking), which are the heart of the guide.

Every chapter has:

- **Concept** sections with ASCII diagrams.
- **On the metal** call-outs: exact byte layouts and struct definitions.
- **Try it** boxes: copy-paste shell commands to reproduce everything.
- **C++ notes**: minutiae that matter specifically to C++ engineers.
- **Pitfalls** and **War stories**: real bugs explained by the mechanics.

---

## Table of contents

### Part 0 — Start Here: the gentle, plain-English track (`00-start-here/`)
*No prerequisites. Read this first if you're new to the subject.*
1. [Read me first](00-start-here/01-read-me-first.md)
2. [The big picture in plain English](00-start-here/02-the-big-picture-plain-english.md)
3. [What is assembly & the assembler?](00-start-here/03-what-is-assembly-and-the-assembler.md)
4. [What is linking?](00-start-here/04-what-is-linking.md)
5. [What are ELF and DWARF?](00-start-here/05-what-are-elf-and-dwarf.md)
6. [Beginner glossary, FAQ & first hands-on](00-start-here/06-beginner-glossary-and-faq.md)

### Part 1 — Foundations (`01-foundations/`)
1. [CPU, registers & the memory model](01-foundations/01-cpu-and-memory.md)
2. [Bits, bytes, endianness & data representation](01-foundations/02-data-representation.md)
3. [The toolchain end to end](01-foundations/03-toolchain-overview.md)

### Part 2 — Assembly language (`02-assembly/`)
1. [x86-64 architecture for the toolchain hacker](02-assembly/01-x86-64-architecture.md)
2. [Instructions, addressing modes & encoding](02-assembly/02-instructions-and-encoding.md)
3. [Assembly syntax, directives & GAS vs NASM](02-assembly/03-syntax-and-directives.md)
4. [Calling conventions & the ABI](02-assembly/04-calling-conventions-abi.md)

### Part 3 — The assembler (`03-assembler/`)
1. [How an assembler works (two passes)](03-assembler/01-how-assemblers-work.md)
2. [Sections, the location counter & symbols](03-assembler/02-sections-and-symbols.md)
3. [Relocations: the assembler's promises](03-assembler/03-relocations.md)

### Part 4 — Object files & ELF (`04-object-files-and-elf/`)
1. [ELF overview & the dual nature of the format](04-object-files-and-elf/01-elf-overview.md)
2. [The ELF header, byte by byte](04-object-files-and-elf/02-elf-header.md)
3. [Sections & the section header table](04-object-files-and-elf/03-sections.md)
4. [Segments & the program header table](04-object-files-and-elf/04-segments.md)
5. [Symbol tables & string tables](04-object-files-and-elf/05-symbols-and-strings.md)
6. [Relocation entries on disk](04-object-files-and-elf/06-relocations-on-disk.md)

### Part 5 — Linking & loading (`05-linking/`)
1. [Static linking from start to finish](05-linking/01-static-linking.md)
2. [Symbol resolution, archives & the ODR](05-linking/02-symbol-resolution.md)
3. [Dynamic linking & shared objects](05-linking/03-dynamic-linking.md)
4. [PLT, GOT & lazy binding](05-linking/04-plt-got.md)
5. [The loader, program startup & `_start`](05-linking/05-loader-and-startup.md)

### Part 6 — DWARF debug info (`06-dwarf/`)
1. [DWARF overview & sections](06-dwarf/01-dwarf-overview.md)
2. [DIEs: the debug information tree](06-dwarf/02-dies-and-debug-info.md)
3. [The line-number program](06-dwarf/03-line-number-program.md)
4. [Call frame information & stack unwinding](06-dwarf/04-cfi-and-unwinding.md)

### Part 7 — Advanced topics (`07-advanced/`)
1. [Thread-local storage (TLS)](07-advanced/01-thread-local-storage.md)
2. [Link-time optimization (LTO)](07-advanced/02-link-time-optimization.md)
3. [C++ name mangling, vtables & COMDAT](07-advanced/03-cpp-abi-mangling.md)
4. [Security mitigations (RELRO, PIE, CFI, stack canaries)](07-advanced/04-security-mitigations.md)
5. [The binary-analysis toolbox](07-advanced/05-tooling-and-debugging.md)

### Part 8 — Hands-on labs (`08-labs/`)
- [Guided labs & exercises](08-labs/exercises.md)

### Reference (`99-reference/`)
- [Glossary](99-reference/glossary.md)
- [Cheat sheets](99-reference/cheatsheets.md)

---

## Platform note

This guide targets the **System V ABI on Linux / x86-64** as the primary
reference platform, because that is where ELF and DWARF live and where the
documentation is richest. Where it matters, we contrast with:

- **AArch64 (ARM64)** — increasingly the default (Apple Silicon, AWS Graviton).
- **macOS / Mach-O** — your local machine if you're on Apple Silicon. The
  *concepts* (symbols, relocations, dynamic linking, DWARF) all transfer; the
  *container format* differs (Mach-O instead of ELF, `dyld` instead of
  `ld.so`). Call-outs note the differences.

To get a genuine ELF/Linux environment on a Mac, use Docker:

```bash
docker run --rm -it -v "$PWD":/work -w /work ubuntu:24.04 bash
# then inside:
apt-get update && apt-get install -y build-essential gdb binutils \
    elfutils dwarfdump file gcc g++ clang lld
```

All "Try it" boxes assume a GNU/Linux toolchain unless stated otherwise.

---

## Conventions used in this guide

| Notation        | Meaning                                              |
|-----------------|------------------------------------------------------|
| `0x...`         | Hexadecimal literal                                  |
| `0b...`         | Binary literal                                       |
| `[expr]`        | Memory dereference (Intel syntax)                    |
| `Elf64_Xxx`     | A C type from `<elf.h>`                              |
| `§N.M`          | Cross-reference to a chapter section                 |
| **Try it ▸**    | A runnable experiment                                |
| **C++ ▸**       | A C++-specific detail                                |
| **Pitfall ▸**   | A common mistake explained mechanically              |

Let's begin. → [Part 1.1: CPU, registers & the memory model](01-foundations/01-cpu-and-memory.md)
