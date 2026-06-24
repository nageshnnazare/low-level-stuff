# 0.3 — What Is Assembly & the Assembler?

Let's gently open the first interesting box. You'll learn what **assembly
language** is, why it exists, and what the **assembler** actually does — all
without needing to memorize any instructions.

---

## 0.3.1 Why computers can't just run your C++ directly

A CPU (the chip that does the work) is astonishingly *simple-minded*. It can only
do very tiny things, like:

- "Copy this number into that storage slot."
- "Add these two numbers."
- "If this is zero, jump to a different instruction."

It has **no idea** what a `for` loop, a `string`, or `printf` is. Those are
human conveniences. So your rich, expressive code has to be broken down into a
long list of these baby steps. **Assembly language is the name for those baby
steps written in a form humans can still read.**

```
   ONE line of C                         BECOMES several assembly steps
   ────────────                          ─────────────────────────────
   sum = a + b;            ─────▶         mov  eax, [a]    ; copy a into a scratch slot
                                          add  eax, [b]    ; add b to it
                                          mov  [sum], eax  ; copy the result into sum
```

Each assembly line is roughly one thing the CPU can do in one go.

---

## 0.3.2 What assembly looks like (don't worry, just look)

Here's a complete tiny function in assembly. You don't need to understand each
line — just notice the *shape*:

```asm
        .text                    ← "the following is code"
        .globl  add_two          ← "the name add_two should be visible to others"
add_two:                         ← a LABEL: this name marks this spot
        lea     eax, [rdi + rsi] ← compute first_arg + second_arg, put in result slot
        ret                      ← return to whoever called us
```

Three kinds of lines, and the distinction matters a lot later:

```
   1. LABELS      end with ":"     →  give a NAME to a location  (add_two:)
   2. INSTRUCTIONS                 →  things the CPU does         (lea, ret)
   3. DIRECTIVES  start with "."   →  instructions to the ASSEMBLER itself,
                                       NOT to the CPU             (.text, .globl)
```

That third category is the surprising one for beginners. A **directive** is not
something the CPU runs — it's a note to the assembler like "put this in the code
section" or "make this name visible to other files." Directives are how the
assembler is told to create the **symbols** and structure that the linker will
later use.

> **Key idea:** a *label* like `add_two:` is how you create a **symbol** (a named
> location). The directive `.globl add_two` is how you say "let other files see
> this symbol." Right there, in two lines, you've set up everything the linker
> needs to connect this function to code in other files.

---

## 0.3.3 Two dialects (so you're not confused later)

The exact same CPU instruction can be written two slightly different ways,
depending on the tool. You'll see both, so just be aware they exist:

```
   AT&T syntax (what GCC/Clang produce by default):
        mov  $5, %eax         ← "move the number 5 into register eax"

   Intel syntax (what most tutorials and Windows tools use):
        mov  eax, 5           ← same thing!
```

The most jarring difference: **the order is reversed.** In AT&T the destination
is on the *right*; in Intel it's on the *left*. Don't memorize this now — just
remember "there are two spellings of the same thing, and the operand order can
flip." (The detailed chapter [2.3](../02-assembly/03-syntax-and-directives.md)
has the full comparison.)

---

## 0.3.4 What the assembler actually does

The **assembler** is the tool that reads assembly text and produces an **object
file** (`.o`) full of machine-code numbers. Its job is mostly a careful,
mechanical translation, but it does three things worth understanding:

### Job 1 — Translate each instruction to numbers

```
   mov eax, 0   ──────▶   b8 00 00 00 00      (these hex numbers ARE the instruction)
   ret          ──────▶   c3
```

That's it for most lines: look up the instruction, write its numeric code.

### Job 2 — Keep track of *where* everything is

As it writes out bytes, the assembler counts its position (like a page counter).
Whenever it hits a label, it records "this name = this position." That's how
symbols get their locations *within this file*.

```
   position 0:  b8 00 00 00 00     ← add_two starts at position 0
   position 5:  c3
   → symbol "add_two" = position 0
```

### Job 3 — Leave blanks (and sticky notes) for what it can't know

Here's the crucial part, and it's exactly the "fill-in-the-blank" idea from 0.2.
When your code refers to something the assembler *can't* locate yet — like
`printf`, which lives in a different file entirely — the assembler:

1. Writes the instruction but leaves the address as **zeros (a blank)**.
2. Attaches a **relocation** (sticky note): "fill this blank in later with the
   real address of `printf`."

```
   call printf                  ← "call the printf function"

   becomes:
        e8 00 00 00 00           ← the "call" instruction + a 4-zero BLANK
              └────┬────┘
                   blank (address unknown to the assembler!)

   plus a sticky note (relocation):
        "at this spot, write the address of the symbol `printf`"
```

The assembler genuinely doesn't know where `printf` will end up — that's not its
job. It just honestly marks "blank, to be filled by the linker." This is the
clean division of labor between the two tools.

---

## 0.3.5 Why "object file" and what's inside it

The assembler's output is an **object file**. "Object" here just means "a piece
of compiled output" — nothing to do with object-oriented programming. It's one
*typeset chapter*, not the whole book.

Inside, an object file is organized into labelled **sections** — think of them
as folders that group similar things:

```
   ┌────────────────────────────────────────────────┐
   │  .text     ← your machine-code instructions    │
   │  .data     ← variables that start with a value │
   │  .bss      ← variables that start at zero      │
   │  .rodata   ← constants (like the text "hello") │
   │  symbol table ← the list of names + locations  │
   │  relocations  ← the sticky notes (blanks list) │
   └────────────────────────────────────────────────┘
```

The two folders to remember: code lives in **`.text`**, and the list of names
lives in the **symbol table**. The linker will read both from every object file.

---

## 0.3.6 See it yourself (optional, but satisfying)

If you have the Linux tools (see 0.1.5), try this. Don't worry about
understanding every detail — the goal is just to *watch the translation happen*.

**Try it ▸**

```bash
# 1. Write a tiny program
cat > tiny.c <<'EOF'
int add_two(int a, int b) { return a + b; }
EOF

# 2. Compile only as far as ASSEMBLY (human-readable), and look at it
gcc -S tiny.c -o tiny.s
cat tiny.s            # you'll see add_two: with a few instructions

# 3. Now assemble into an OBJECT FILE (numbers) and confirm its type
gcc -c tiny.c -o tiny.o
file tiny.o           # says "ELF ... relocatable" = an object file

# 4. Disassemble it: turn the numbers BACK into readable instructions
objdump -d tiny.o     # shows the hex bytes AND the instructions side by side
```

That last command is wonderful for beginners: it shows the raw numbers on the
left and the human-readable instruction on the right, so you can see they're two
views of the same thing.

**Try it ▸** Now see a "blank + sticky note" in real life:

```bash
cat > callp.c <<'EOF'
#include <stdio.h>
void greet(void) { printf("hi\n"); }
EOF
gcc -c callp.c -o callp.o
objdump -dr callp.o    # the -r adds the relocations (sticky notes) inline
```

Look for a line like `call` followed by zeros, with a note mentioning `printf`
nearby. That's the blank and its relocation, exactly as described above.

---

## 0.3.7 What you now understand

- A CPU only does tiny steps; **assembly** is those steps in human-readable form,
  and the **compiler** is what breaks your code down into them.
- Assembly lines are **labels** (names for locations → these become *symbols*),
  **instructions** (what the CPU does), and **directives** (notes to the
  assembler, like `.text` and `.globl`).
- The **assembler** translates instructions to numbers, records where each name
  lives, and — crucially — **leaves blanks with relocation notes** for anything
  it can't locate yet.
- Its output is an **object file** (`.o`), organized into **sections** like
  `.text` (code) plus a **symbol table** (names) and **relocations** (blanks).

You've now got the first half. Next comes the part you specifically asked
about — what the linker does with all these chapters.

→ [0.4 — What is linking? (the heart of it)](04-what-is-linking.md)
