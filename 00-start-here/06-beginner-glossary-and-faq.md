# 0.6 — Beginner Glossary, FAQ & Your First Hands-On

Final stop on the gentle track. This chapter gives you (1) a plain-English
glossary you can return to whenever a word confuses you, (2) answers to the
questions almost every beginner asks, and (3) one guided experiment that lets you
*watch the entire pipeline* on your own screen.

---

## 0.6.1 Plain-English glossary

These are deliberately *informal* one-liners with analogies. For the precise
definitions, see [99-reference/glossary.md](../99-reference/glossary.md) later.

| Word | In plain English | Book analogy |
|------|------------------|--------------|
| **Source code** | The text you write (C/C++). | Your handwritten draft. |
| **Compiler** | Turns your code into assembly (tiny CPU steps). | The editor rewriting prose into a checklist. |
| **Assembly** | CPU instructions in human-readable form. | The checklist of tiny actions. |
| **Assembler** | Turns assembly into machine-code numbers. | The typesetter setting final letters. |
| **Machine code** | The raw numbers the CPU actually runs. | The printed letters on the page. |
| **Object file (`.o`)** | One compiled piece, not yet a whole program. | One typeset chapter. |
| **Section** | A labelled compartment in a file (`.text` = code). | A folder grouping similar pages. |
| **Symbol** | A name for a function/variable + where it lives. | An index entry: "printf … p.500". |
| **Defined symbol** | "I have this thing, here it is." | A chapter that contains the recipe. |
| **Undefined symbol** | "I need this thing; someone else has it." | "See the recipe elsewhere." |
| **Relocation** | A fill-in-the-blank note: "put X's address here later." | "See page ___" waiting to be filled. |
| **Linker** | Combines pieces + connects every need to a definition. | The bindery assembling the book. |
| **Address** | A number identifying a spot in memory. | A page number. |
| **Library** | Reusable code someone else wrote (e.g. `printf`). | A reference book you quote from. |
| **Static linking** | Copy the library into your program. | Reprint the quote fully in your book. |
| **Dynamic linking** | Borrow the library at run time. | "See the City Library, shelf 3." |
| **Shared library (`.so`)** | A library borrowed at run time. | The book at the City Library. |
| **Loader** | Puts the finished program in memory and starts it. | The reader opening the book. |
| **Process** | A program that is currently running. | The book being actively read. |
| **ELF** | The file format for programs on Linux. | The standardized book format. |
| **DWARF** | Optional notes mapping machine details to your source. | Margin notes: "this = Chapter 1, line 5." |
| **Debugger** | A tool to inspect/step through a running program. | A reading assistant who can pause and explain. |
| **Strip** | Remove debug info / symbols to shrink the file. | Tearing out the margin notes and index. |

---

## 0.6.2 Frequently asked beginner questions

**Q: I just type `gcc hello.c`. Do I ever use the assembler and linker directly?**
Usually no. `gcc` (or `g++`/`clang`) is a *manager* that runs the compiler,
assembler, and linker for you and deletes the in-between files. This guide teaches
you to *see inside* that manager — which is what makes you good at fixing build
problems.

**Q: What's the difference between compiling and linking?**
*Compiling* (which includes assembling) turns each source file into its own
object file, independently. *Linking* combines all those object files and
libraries into one finished program and connects the references. Compiling is
"per-file"; linking is "bring it all together." Errors that mention specific code
mistakes are usually compile errors; errors about `undefined reference` or
`multiple definition` are *link* errors.

**Q: Why do I get `undefined reference to 'foo'`?**
You *used* `foo` (a function or variable) but no object file or library you gave
the linker actually *defines* it. Either you forgot to include the file/library
that has `foo`, or you misspelled it, or (in C/C++) you declared it but never
wrote its body. See [0.4.4](04-what-is-linking.md).

**Q: Why does the order of files/libraries on the command line matter?**
The linker reads left to right and, for libraries, only pulls in what's *already*
been requested. So a library must come *after* the files that use it:
`gcc main.o -lm` works, but `gcc -lm main.o` often fails. (Full explanation:
[5.2](../05-linking/02-symbol-resolution.md).)

**Q: My program built fine but won't run — `cannot open shared object file`?**
That's a *dynamic linking* problem at run time: your program borrows a library
that the system can't find when starting up. The build succeeded because the
borrowing is deferred to run time. See [0.4.6](04-what-is-linking.md) and
[5.3](../05-linking/03-dynamic-linking.md).

**Q: Do I need to learn assembly to be a good programmer?**
No — but being able to *read* a little (not write it) is a superpower for
understanding performance, debugging tricky crashes, and demystifying the tools.
This guide teaches reading, not writing.

**Q: What's a "register"?**
A tiny, ultra-fast storage slot *inside* the CPU. The CPU can't do math on data
in main memory directly; it loads values into registers, works on them, and
writes results back. You'll see names like `rax`, `rdi` in assembly — those are
registers. ([1.1](../01-foundations/01-cpu-and-memory.md).)

**Q: What does `-g` do, and what does "strip" mean?**
`-g` tells the compiler to include **DWARF** debug info so debuggers can show your
source lines and variables. *Stripping* removes that info (and other names) to
make the file smaller — common for shipping finished software. See
[0.5.3](05-what-are-elf-and-dwarf.md).

**Q: Is this Linux-only? I'm on a Mac / Windows.**
The *ideas* are universal. The exact file formats and tool names differ: macOS
uses "Mach-O" files and a loader called `dyld`; Windows uses "PE" files and
`.dll` libraries. We teach the Linux versions (ELF/DWARF) because they're the
most open and best-documented, and everything transfers. Use Docker or WSL (see
[0.1.5](01-read-me-first.md)) to follow along exactly.

**Q: This is a lot. Did I miss something if I don't get it all?**
No. This is deep material that professionals study for years. Re-reading is
normal. If you understood the one-sentence summary of the linker in
[0.4.5](04-what-is-linking.md), you're in great shape.

---

## 0.6.3 Your first hands-on: watch the whole pipeline

This single guided experiment shows you every stage from your earlier reading,
all in a row. Run each block and read the comment explaining what you just saw.
(Need the tools? See [0.1.5](01-read-me-first.md).)

**Step 1 — Write a tiny two-file program.**

```bash
cat > greet.c <<'EOF'
#include <stdio.h>
const char* who(void);                 /* a promise: who() exists somewhere */
int main(void){ printf("hello, %s\n", who()); return 0; }
EOF

cat > who.c <<'EOF'
const char* who(void){ return "world"; }   /* the definition of who() */
EOF
```

**Step 2 — Stop after COMPILE+ASSEMBLE: make object files, don't link.**

```bash
gcc -c greet.c -o greet.o
gcc -c who.c   -o who.o
file greet.o who.o          # both say "ELF ... relocatable" = object files (chapters)
```

**Step 3 — Look at the symbols (names) in each piece.**

```bash
nm greet.o
#   ...  U who        ← 'U' = UNDEFINED: greet NEEDS who (a promise to fill)
#   ...  U printf     ← greet also needs printf (from the C library)
#   ...  T main       ← 'T' = defined code: greet HAS main
nm who.o
#   ...  T who        ← who.o DEFINES who
```

You're literally seeing the "needs" and "haves" from [0.4.4](04-what-is-linking.md).

**Step 4 — See a blank + its relocation note.**

```bash
objdump -dr greet.o | grep -A1 -i 'call'
# Near a 'call' you'll see zeros (the BLANK) and a note naming `who`/`printf`
# (the RELOCATION). Exactly the "fill-in-later" idea.
```

**Step 5 — LINK them into a finished program, and run it.**

```bash
gcc greet.o who.o -o greet
./greet                     # prints: hello, world
```

The linker matched `greet`'s need for `who` to `who.o`'s definition, filled the
blanks, and produced a runnable program.

**Step 6 — Prove the linker is doing the matching: remove a piece.**

```bash
gcc greet.o -o broken       # ERROR: undefined reference to `who'
```

You left out `who.o`, so nobody defines `who`. Now this error reads like an old
friend, not a mystery.

**Step 7 — See dynamic linking (the borrowing).**

```bash
ldd greet                   # lists libc — that's where printf is borrowed from
file greet                  # "dynamically linked"
```

**Step 8 (optional) — Add debug info and map an address to a source line.**

```bash
gcc -g greet.c who.c -o greet_dbg
addr2line -e greet_dbg -f -i 0x$(nm greet_dbg | awk '/ T main/{print $1}' | tail -c 5) 2>/dev/null
# Thanks to DWARF (-g), this prints the function name and source line.
```

If you ran all eight steps, you've personally observed: compiling to object
files, symbols (needs vs haves), a relocation blank, successful linking, an
`undefined reference` error, dynamic borrowing, and DWARF. That's the whole
guide in miniature.

---

## 0.6.4 You're ready for the deep dive

You now have the full mental model:

```
   code → [compile] → [assemble] → object files (with blanks + symbols)
        → [link] (match needs↔haves, fill blanks) → program
        → [load] → running process
   ...all stored in ELF files, with optional DWARF notes for debugging.
```

When you're ready to see the exact bytes and mechanics behind each idea, continue
to the main guide. A suggested order from here:

1. [Part 1 — Foundations](../01-foundations/01-cpu-and-memory.md): the CPU,
   memory, and number basics underneath everything.
2. [Part 3 — The Assembler](../03-assembler/01-how-assemblers-work.md): the
   precise version of [0.3](03-what-is-assembly-and-the-assembler.md).
3. [Part 5 — Linking](../05-linking/01-static-linking.md): the precise version of
   [0.4](04-what-is-linking.md).
4. Then Parts 4 (ELF), 6 (DWARF), and 7 (advanced) as your curiosity leads.

Take your time, run the experiments, and revisit this Start Here track whenever a
later chapter feels too dense. Welcome to the field — you're going to enjoy
finally seeing how it all really works.

← Back to [the guide's main README](../README.md)
