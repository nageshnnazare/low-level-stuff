# 0.1 — Read Me First (Absolute Beginner's Welcome)

> **You don't need any background to read this part.** No assembly, no
> command-line wizardry, no computer-science degree. If you can write a "hello
> world" program in *any* language, you're ready.

This **Start Here** track exists to give you the *intuition* first — using
everyday analogies and pictures — so that when you reach the detailed,
byte-level chapters later, they feel like "oh, I already understand the idea,
now I'm just seeing the exact mechanics."

---

## 0.1.1 What this whole guide is really about

When you write a program and run it, a hidden assembly line of tools quietly
turns your human-friendly text into something a computer chip can actually
execute. This guide is about **that hidden assembly line**.

```
   You write this:                   The computer runs this:
   ┌────────────────────┐            ┌──────────────────────────┐
   │ int main() {       │            │ 01010101 01001000 ...    │
   │   printf("hi\n");  │   ????→    │ (raw numbers the CPU     │
   │ }                  │            │  understands)            │
   └────────────────────┘            └──────────────────────────┘
            ▲                                     ▲
            │                                     │
        what YOU                          what the MACHINE
        understand                          understands
```

The big question this guide answers: **what happens in that "????" arrow?**

The short answer is: several tools, each doing one job, passing their work to
the next. The two stars of the show — and the ones you asked about — are:

- The **assembler** — turns human-readable CPU instructions into raw numbers.
- The **linker** — takes the separate pieces of your program (and of libraries
  others wrote) and stitches them into one finished, runnable program.

We'll also meet the **compiler** (which comes before the assembler), the
**loader** (which starts your program running), and the **file formats** (ELF
and DWARF) that hold everything together.

---

## 0.1.2 Why should you care? (Real things this explains)

Understanding this pipeline turns scary, mysterious errors into things that make
sense. By the end you'll understand *why* each of these happens:

```
   "undefined reference to `foo'"        ← the linker couldn't find a piece
   "multiple definition of `foo'"        ← two pieces claim the same name
   "cannot open shared object file"      ← a library is missing at run time
   "version `GLIBC_2.34' not found"      ← built against a newer system library
   "undefined reference to vtable for X" ← a classic C++ trap
   "segmentation fault"                  ← your program touched forbidden memory
```

These aren't random. Each one comes from a specific, understandable step in the
pipeline. Right now they might feel like the computer yelling at you in a
foreign language. Soon they'll feel like helpful, specific clues.

---

## 0.1.3 How to use this Start Here track

Read these short chapters **in order**. They're meant to be read like a story,
not skimmed like a reference:

```
  0.1  Read me first              ← you are here
  0.2  The big picture in plain English   (the whole journey, with analogies)
  0.3  What is assembly & the assembler?  (your text → CPU numbers)
  0.4  What is linking? (the heart)       (combining the pieces) ★ extra detail
  0.5  What are ELF and DWARF?            (the file formats, gently)
  0.6  Beginner glossary, FAQ & first hands-on experiment
```

Then, when you feel comfortable with the *ideas*, move on to **Part 1
(Foundations)** and beyond, where we open up the actual bytes. The later parts
are denser — but you'll be ready.

---

## 0.1.4 A few promises about the style here

1. **Every new word is explained the first time it appears.** If we say
   "symbol," we tell you what a symbol is, in plain words, right there.
2. **Analogies first, precision later.** In this track we happily say "it's
   *like* a phone book." The detailed parts will make it exact.
3. **You can run everything.** The "Try it yourself" boxes are copy-paste
   commands. You don't have to — but seeing it on your own screen makes it
   click. (If you're on a Mac, see the note below.)
4. **It's okay not to get everything at once.** This material is genuinely deep.
   Re-reading is normal and expected.

---

## 0.1.5 One setup note (so the experiments work)

The hands-on examples use Linux tools (because that's where these formats, ELF
and DWARF, live). You have three easy options:

- **You're on Linux already** → great, everything just works.
- **You're on a Mac** → the *ideas* are identical, but Apple uses slightly
  different tools and file formats. To follow along exactly, run a tiny Linux
  "sandbox" with Docker:

  ```bash
  docker run --rm -it -v "$PWD":/work -w /work ubuntu:24.04 bash
  # then, once inside the prompt:
  apt-get update && apt-get install -y build-essential gdb binutils file
  ```

- **You're on Windows** → use WSL (Windows Subsystem for Linux) or the same
  Docker command.

Don't worry if you skip the setup for now — you can just read. The pictures and
explanations stand on their own.

---

## 0.1.6 The one mental model to carry with you

If you remember nothing else, remember this: **building a program is an assembly
line where each tool does one small job and hands the result to the next.**

```
   your code → [compiler] → [assembler] → [linker] → a program → [loader] → running!
                  │             │             │                      │
              "translate    "turn into    "combine the          "place it in
               to CPU         raw           pieces into          memory and
               steps"         numbers"      one whole"           press start"
```

Everything else is just zooming into one of those boxes. Let's start zooming.

Next: [0.2 — The big picture in plain English](02-the-big-picture-plain-english.md)
