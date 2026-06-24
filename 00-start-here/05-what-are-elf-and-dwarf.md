# 0.5 — What Are ELF and DWARF?

You've met the *tools* (compiler, assembler, linker, loader). Now let's gently
meet the *file formats* they all read and write. Two names you'll see constantly:
**ELF** and **DWARF**. They sound intimidating; they're not.

---

## 0.5.1 A "format" is just an agreed-upon layout

A file is just a long row of bytes (numbers). For different programs to agree on
what those bytes *mean*, they follow a **format** — a shared rulebook for the
layout.

```
   A "format" answers: "if I open this file, where do I find each thing?"

   Like a standardized form:
   ┌─────────────────────────────┐
   │ Name:  [______________]     │  ← everyone knows the name goes HERE
   │ Date:  [____________]       │  ← and the date goes HERE
   │ ...                         │
   └─────────────────────────────┘
```

You already rely on formats every day: `.jpg` is a format for images, `.mp3` for
audio, `.pdf` for documents. ELF and DWARF are just formats too — for programs
and for debug information.

---

## 0.5.2 ELF — the container for programs

**ELF** stands for **Executable and Linkable Format**. It's the standard file
format on Linux (and many other systems) for:

```
   • object files      (.o)   ← the assembler's output
   • executables              ← the finished program you run
   • shared libraries  (.so)  ← libraries borrowed at run time
```

So *every* file we've discussed — object files, the final program, libraries —
is an ELF file. When you ran `file tiny.o` earlier and saw "ELF ...
relocatable," that was ELF announcing itself.

### What's inside an ELF file (the friendly version)

Think of an ELF file as a box with a **label on the lid** and several
**compartments** inside:

```
   ┌────────────────────────────────────────────────┐
   │  ELF HEADER  (the label on the lid)            │
   │   - "I am an ELF file"  (a magic marker)       │
   │   - "I'm 64-bit, for x86 CPUs"                 │
   │   - "I'm an executable" (or object, or library)│
   │   - "start running at this spot"               │
   ├────────────────────────────────────────────────┤
   │  .text       compartment: the code             │
   │  .data       compartment: variables with values│
   │  .bss        compartment: variables (start 0)  │
   │  .rodata     compartment: constants            │
   │  symbol table: the list of names + locations   │
   │  relocations:  the fill-in-the-blank notes     │
   └────────────────────────────────────────────────┘
```

The very first bytes of every ELF file are a tiny signature so any tool can
instantly recognize it:

```
   The first 4 bytes are ALWAYS:   7F 45 4C 46
                                       └ 'E' 'L' 'F' in text!
   So an ELF file literally starts by spelling out "ELF".
```

**Try it ▸** Peek at those first bytes yourself:

```bash
gcc tiny.c -c -o tiny.o 2>/dev/null || echo 'int x;' | gcc -x c -c - -o tiny.o
xxd tiny.o | head -1
# You'll see: 7f45 4c46 ...  →  ".ELF..." on the right side
```

That's genuinely the start of every program on a Linux machine. Not so scary.

### The one ELF idea worth knowing early: two ways to view the file

ELF cleverly describes the *same* file from two angles, for two different
readers:

```
   FOR THE LINKER:                    FOR THE LOADER (run time):
   "here are the fine-grained          "here are the big chunks to copy into
    folders: .text, .data, ..."         memory, and what permission each gets"
   (called SECTIONS)                   (called SEGMENTS)
```

Don't worry about the details now — just know that "sections" (linker's view)
and "segments" (loader's view) are two labels for organizing the same bytes. The
detailed [Part 4](../04-object-files-and-elf/01-elf-overview.md) makes this
precise; for now, "sections = for building, segments = for running" is plenty.

---

## 0.5.3 DWARF — the format for debug information

Here's a puzzle. After all the translation, the running program is just numbers
and addresses. So how does a **debugger** (a tool that helps you find bugs) know
that address `0x1149` is your function `main`, on line 5 of `main.c`, and that
the value in some CPU slot is your variable named `count`?

The answer: extra notes, written in a format called **DWARF**.

```
   The program at run time knows ONLY:        DWARF adds the human meaning:
   ┌───────────────────────────┐               ┌────────────────────────────────┐
   │ address 0x1149            │   ◀──DWARF──▶ │ "that's main(), main.c line 5" │
   │ a CPU slot holds 42       │               │ "that slot is the variable     │
   │ ...                       │               │  `count`, which is an int"     │
   └───────────────────────────┘               └────────────────────────────────┘
```

(Yes — **ELF** and **DWARF** is a deliberate fantasy-themed pun by their
creators. It's the only joke in the whole field, enjoy it.)

DWARF is what makes this possible:

```
   • Setting a breakpoint at "main.c line 10"      ← needs line ↔ address mapping
   • Seeing variable names and values in a debugger ← needs variable info
   • A crash report saying "crashed at main.c:42"   ← needs the same mapping
   • Stepping through your code line by line         ← needs all of the above
```

### Where DWARF lives, and why it's optional

DWARF information is stored in extra compartments inside the ELF file, all named
`.debug_*` (like `.debug_info`, `.debug_line`). Two important facts for
beginners:

```
   1. It's only added if you ASK for it (the -g flag):
         gcc        main.c -o app      ← no debug info (smaller)
         gcc -g     main.c -o app      ← WITH debug info (debuggable)

   2. It is NOT loaded when the program runs — it just sits in the file for
      debuggers to read. So it never slows your program down; it only makes
      the file on disk bigger.
```

This is why advice for debugging is always "compile with `-g`," and why released
software is often **stripped** (debug info removed) to save space — at the cost
of harder debugging.

**Try it ▸** See debug info appear and disappear:

```bash
cat > dbg.c <<'EOF'
#include <stdio.h>
int main(void){ int count = 42; printf("%d\n", count); return 0; }
EOF

gcc     dbg.c -o app_nodebug      # no -g
gcc -g  dbg.c -o app_debug        # with -g

ls -l app_nodebug app_debug       # the -g one is bigger (carries DWARF)

# This works ONLY when debug info is present:
addr2line -e app_debug -f 0x$(nm app_debug | awk '/ T main/{print $1}' | tail -c 5) 2>/dev/null
# → prints the function and source line, thanks to DWARF
```

---

## 0.5.4 How it all fits together

```
   ┌─────────────────────────── one ELF file ─────────────────────────────┐
   │  ELF header        "I'm a program, here's how to read me"            │
   │  .text             your code (the assembler/linker put it here)      │
   │  .data / .bss      your variables                                    │
   │  symbol table      names → locations (used by the linker)            │
   │  relocations       fill-in-the-blank notes (used by the linker)      │
   │  .debug_* (DWARF)  optional human-meaning notes (used by debuggers)  │
   └──────────────────────────────────────────────────────────────────────┘
          ▲                                          ▲
          │ written by the assembler & linker        │ written if you pass -g
          │ read by the linker & loader              │ read by debuggers
```

- **ELF** = the box that holds your program and all its parts.
- **DWARF** = optional notes inside that box translating machine details back to
  your source code, for debugging.

---

## 0.5.5 What you now understand

- A **format** is just an agreed-upon layout so different tools interpret the
  same bytes the same way (like `.jpg` or `.pdf`).
- **ELF** is the format for object files, executables, and shared libraries on
  Linux. It has a **header** (the label) and **compartments** (code, data,
  symbol table, relocations). Every ELF file literally starts with the bytes that
  spell "ELF."
- ELF describes the same file two ways: **sections** (for building/linking) and
  **segments** (for running/loading).
- **DWARF** is the optional format for **debug information** — the notes that let
  a debugger map addresses back to your function names, variables, and source
  lines. You get it with `-g`, and it's never loaded at run time.

You've now met every actor and every format. The last chapter wraps up with a
plain-English glossary, answers to common beginner questions, and a single guided
experiment that shows the whole pipeline at once.

→ [0.6 — Beginner glossary, FAQ & first hands-on](06-beginner-glossary-and-faq.md)
