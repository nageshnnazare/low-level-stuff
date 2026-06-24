# 0.2 — The Big Picture in Plain English

Before any details, let's understand the *whole journey* of a program with a
single story you already understand: **publishing a book**. Hold this analogy in
your head — we'll reuse it constantly.

---

## 0.2.1 The "publishing a book" analogy

Imagine you and some friends are writing a book together.

```
   WRITING A BOOK                          BUILDING A PROGRAM
   ──────────────                          ──────────────────
   You write chapters in plain English  ≈  You write code in C/C++
   An editor turns your draft into the  ≈  The COMPILER turns your code into
     publisher's house style                 the CPU's instruction style
   A typesetter sets it into final      ≈  The ASSEMBLER turns those instructions
     printed letters and page numbers        into the exact numbers (machine code)
   The bindery gathers all the chapters ≈  The LINKER gathers all the pieces
     (yours + quoted material) into one      (your code + libraries) into one
     bound book                              finished program
   A reader opens the book and reads    ≈  The LOADER opens the program and runs it
```

That's the entire pipeline. Five roles. Let's meet each one properly.

---

## 0.2.2 The five stages, one at a time

### Stage 1 — The compiler (the "editor")

You write **source code** — text that's friendly for humans:

```c
int main() {
    printf("hello\n");
    return 0;
}
```

The **compiler** reads this and rewrites it into **assembly** — a much more
primitive, step-by-step set of instructions written for the CPU. Where you wrote
one line, the compiler might produce ten tiny steps, because the CPU only
understands very small operations ("copy this number here," "add these two,"
"jump to there").

> Think of the compiler as an editor who takes your flowing prose and rewrites
> it as a numbered checklist of tiny, unambiguous actions.

### Stage 2 — The assembler (the "typesetter")

Assembly is *readable*, but the CPU still can't run it directly — the CPU only
understands **numbers**. The **assembler** does a fairly mechanical translation:
each assembly instruction becomes its exact numeric code.

```
   assembly (human-ish)         machine code (pure numbers, shown in hex)
   ────────────────────         ─────────────────────────────────────────
   mov eax, 0           ──────▶  b8 00 00 00 00
   ret                  ──────▶  c3
```

The result is an **object file** (a `.o` file). Think of it as a *typeset
chapter*: the words are now in final printed form, **but it's not a whole book
yet** — it's just one chapter, and it might contain phrases like "see the recipe
on page ???" where the page number isn't known yet.

### Stage 3 — The linker (the "bindery")

Real programs are built from **many** pieces:

- Your own code, split across several files.
- **Libraries** — code other people already wrote (like `printf`, which lives in
  the system's "C library").

The **linker** gathers all these chapters into one bound book. Its two big jobs:

1. **Put the chapters in order** and number all the pages (assign every piece a
   place in memory).
2. **Fill in all the "see page ???" cross-references.** When your chapter said
   "call `printf`," the linker finally knows *which* page `printf` is on and
   writes that number in.

```
   your chapter      library chapter        the finished, bound book
   ┌───────────┐      ┌───────────────┐       ┌───────────────────────────┐
   │ ...call   │      │ printf:       │       │ Ch.1 (yours): call p.500  │
   │  printf   │  +   │  <the actual  │  ───▶ │ ...                       │
   │  (p. ???) │      │   code>       │       │ Ch.7 (printf): p.500 ...  │
   └───────────┘      └───────────────┘       └───────────────────────────┘
              "???" gets filled in ───────────────▶ "p.500"
```

This is why you asked about it, and it's the heart of the guide — so chapter 0.4
is entirely about linking, gently.

### Stage 4 — The loader (the "reader opening the book")

The finished program sits on disk as a file. When you run it, the **loader**
(part of the operating system) copies it into the computer's memory, does a few
last-minute fix-ups, and tells the CPU "start reading at the beginning." Now your
program is *alive* — a running **process**.

### Stage 5 — Running

The CPU executes your instructions one after another, and you see `hello` on the
screen.

---

## 0.2.3 The whole journey in one picture

```
   hello.c   ── your source code (text you wrote)
      │
      │  ① COMPILER  ("rewrite as tiny CPU steps")
      ▼
   hello.s   ── assembly (readable CPU instructions)
      │
      │  ② ASSEMBLER ("turn each step into numbers")
      ▼
   hello.o   ── object file (one typeset chapter, with "see ???" gaps)
      │
      │  ③ LINKER    ("combine all chapters + fill in the ??? references")
      ▼
   hello     ── the finished program (a complete, bound book)
      │
      │  ④ LOADER    ("open it, place in memory, press start")
      ▼
   a running program saying "hello"
```

**Good news for beginners:** normally you never run these four tools by hand.
When you type `gcc hello.c -o hello`, the `gcc` command is a **manager** that
quietly runs all four for you and throws away the intermediate files. This guide
just teaches you to *see inside* that manager.

---

## 0.2.4 Two words you'll hear constantly: "symbol" and "relocation"

Almost everything in linking comes down to two simple ideas. Let's name them
now, in plain English, using the book analogy.

### A "symbol" is a name with a location

A **symbol** is just **a name the program uses for something** — a function or a
variable — plus the information of *where it lives*.

```
   "printf"  is a symbol  → it names a function, located at some page
   "main"    is a symbol  → it names YOUR function, located at some page
   "answer"  is a symbol  → it names a variable, located at some page
```

In the book analogy, a symbol is like an **index entry**: "printf ........ p.500".

There are two flavors of symbol, and the difference is the whole reason linking
exists:

```
   DEFINED symbol  : "I have this here, and it's on page 500."   (a definition)
   UNDEFINED symbol: "I need this, but I don't have it — someone else does."
                       (a reference / a promise to fill in later)
```

The linker's main job is to **match every UNDEFINED symbol to a DEFINED one**. If
it can't, you get the famous `undefined reference` error — literally "you used a
name, but nobody defined it."

### A "relocation" is a fill-in-the-blank note

A **relocation** is a little sticky note that says: *"There's a blank here.
Once you know the real page number for symbol X, write it in this exact spot."*

```
   In your typeset chapter:
   "...now call the function at page [____]"   ← a blank
                                  ▲
                                  └ a relocation note attached:
                                    "fill this blank with printf's page number"
```

The assembler leaves the blanks and writes the sticky notes (because *it* doesn't
know the final page numbers yet). The linker reads the notes and fills the blanks
(because by then it *does* know). That handoff — **assembler leaves blanks +
notes; linker fills them in** — is the single most important idea in this whole
guide.

---

## 0.2.5 Static vs dynamic: "bind the library in" or "borrow it each time"

One more big-picture choice. Library code (like `printf`) can be combined with
your program in two ways:

```
   STATIC linking:  copy the library's chapters INTO your book.
       → Your book is bigger, but completely self-contained.
       → Like printing the full text of every quote inside your book.

   DYNAMIC linking: just write "see the City Library, shelf 3" and fetch it
       when the reader needs it.
       → Your book is smaller; many books share one copy at the City Library.
       → BUT if the City Library is missing that shelf, the reader is stuck
         ("cannot open shared object file").
```

```
   STATIC:                          DYNAMIC:
   ┌──────────────────┐              ┌──────────┐      ┌────────────────┐
   │ your code        │              │ your code│ ───▶ │ shared library │
   │ + a copy of      │              │ (small)  │      │ (one shared    │
   │   printf's code  │              └──────────┘      │  copy on disk) │
   └──────────────────┘             ┌──────────┐ ───▶  │                │
   (self-contained, larger)         │ other app│       └────────────────┘
                                    └──────────┘  (everyone borrows the same one)
```

Most programs today are **dynamically** linked (smaller, libraries can be updated
once for everybody). We'll explore both, but just knowing the two styles exist
puts you ahead.

---

## 0.2.6 What you now understand

You can now tell the story of how a program is built:

1. The **compiler** turns your code into assembly (tiny CPU steps).
2. The **assembler** turns assembly into machine-code numbers in an **object
   file**, leaving **blanks** for things it doesn't know yet, with **relocation**
   notes attached.
3. The **linker** combines all object files and libraries, matches every
   **undefined symbol** to a **definition**, and fills in the blanks.
4. The **loader** places the finished program in memory and starts it.
5. Libraries can be **statically** baked in or **dynamically** borrowed at run
   time.

That's genuinely the core of the entire field. Everything after this is detail.

Next, we zoom into the first half — assembly and the assembler.

→ [0.3 — What is assembly & the assembler?](03-what-is-assembly-and-the-assembler.md)
