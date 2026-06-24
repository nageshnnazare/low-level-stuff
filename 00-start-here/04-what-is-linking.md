# 0.4 — What Is Linking? (The Heart of It)

This is the chapter you came for. The linker is, for most beginners, the most
*mysterious* tool — and also the one that produces the most confusing errors. By
the end of this chapter it will feel simple and obvious. We'll go slowly and use
lots of pictures.

---

## 0.4.1 Why do we even need a linker?

Two reasons, both about **splitting work into pieces**.

### Reason 1: Your own program is split across files

Real programs aren't one giant file. You split them up — `math.c`, `ui.c`,
`main.c` — so they're easier to manage and faster to rebuild. Each file is
compiled *separately* into its own object file. But code in `main.c` often needs
to call a function defined in `math.c`. Someone has to **connect** them.

```
   main.c  ──compile──▶ main.o   (uses a function add(), but doesn't have it)
   math.c  ──compile──▶ math.o   (defines add(), but nobody here calls it)
                            │
                            ▼
                      LINKER connects "main needs add" ⟷ "math has add"
```

### Reason 2: You use libraries other people wrote

When you call `printf`, you didn't write it — it lives in the system's **C
library**. Your object file just has a *reference* ("I need printf"). The linker
connects that reference to the real `printf` code in the library.

So the linker exists to **combine separately-built pieces into one whole, and
connect every reference to its definition.** That's the entire idea.

---

## 0.4.2 The linker's two big jobs

Everything the linker does falls into two buckets. Keep these two words in mind
the whole time: **layout** and **resolution**.

```
   ┌───────────────────────────────────────────────────────────────────┐
   │  JOB 1 — LAYOUT:                                                  │
   │     Gather all the pieces, stack them together, and give every    │
   │     piece a final address (a "page number" in memory).            │
   │                                                                   │
   │  JOB 2 — RESOLUTION (+ filling blanks):                           │
   │     Match every "I need X" to a "here is X", then go back and fill│
   │     in all the blanks the assembler left, now that addresses are  │
   │     known.                                                        │
   └───────────────────────────────────────────────────────────────────┘
```

Let's take them one at a time.

---

## 0.4.3 Job 1: Layout — stacking the pieces and numbering the pages

Each object file contains sections (folders) like `.text` (code) and `.data`
(variables). The linker takes the same-named folders from every file and **stacks
them together**, then decides where in memory the whole stack will live.

```
   main.o            math.o                The combined program
   ┌────────┐        ┌────────┐            ┌──────────────────────┐
   │ .text  │        │ .text  │            │ .text:               │
   │ (code  │   +    │ (code  │   ───────▶ │   main's code        │
   │  for   │        │  for   │            │   math's code        │  ← stacked!
   │  main) │        │  add)  │            │ .data:               │
   ├────────┤        ├────────┤            │   main's variables   │
   │ .data  │        │ .data  │            │   math's variables   │
   └────────┘        └────────┘            └──────────────────────┘
```

Then it assigns real **addresses**. An address is just a number identifying a
location in memory — exactly like a page number in our book.

```
   After layout, the linker knows things like:
       main's code starts at address 0x1130
       add's code  starts at address 0x1150
       a variable "answer" lives at address 0x4010
```

Before this step, nobody knew these numbers. *This* is the step that finally
turns "somewhere in this file, at offset 5" into "address 0x1155 in the final
program." And once we know real addresses, we can do Job 2.

> **Why couldn't the assembler do this?** Because when the assembler processed
> `main.o`, it had no idea `math.o` even existed, let alone how big it was. Only
> the linker sees *all* the pieces at once, so only the linker can lay them out.

---

## 0.4.4 Job 2: Resolution — matching "needs" to "haves"

Now the linker builds one big master list of every **symbol** (named thing) from
every file, marking each as either **defined** ("I have it, here's where") or
**undefined** ("I need it, find it for me").

```
   Master symbol list while linking main.o + math.o:
   ┌──────────┬───────────────┬──────────────────────────┐
   │ name     │ status        │ where                    │
   ├──────────┼───────────────┼──────────────────────────┤
   │ main     │ DEFINED       │ main.o, address 0x1130   │
   │ add      │ DEFINED       │ math.o, address 0x1150   │
   │ printf   │ UNDEFINED     │ → look in the C library  │
   └──────────┴───────────────┴──────────────────────────┘
```

The linker walks the list and **matches every UNDEFINED to a DEFINED of the same
name**:

```
   main.o says: "I need `add`"   (undefined)
   math.o says: "I have `add`"   (defined at 0x1150)
                    │
                    └──── MATCH! main's call to add now points to 0x1150
```

This matching is the famous part. Two things can go wrong, and now you'll
understand both errors completely:

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  Nobody DEFINES a name you NEED                                      │
   │     →  "undefined reference to `foo'"                                │
   │     →  meaning: you used foo, but no file (and no library) has it.   │
   │     →  fix: add the file/library that defines foo to the build.      │
   ├──────────────────────────────────────────────────────────────────────┤
   │  TWO files DEFINE the same name                                      │
   │     →  "multiple definition of `foo'"                                │
   │     →  meaning: the linker doesn't know which one you mean.          │
   │     →  fix: remove one definition (or make it `static`/inline).      │
   └──────────────────────────────────────────────────────────────────────┘
```

That's it. The two scariest linker errors are just "I couldn't find a
definition" and "I found too many." Specific, sensible, fixable.

---

## 0.4.5 Filling in the blanks (relocations, revisited)

Remember the **blanks** the assembler left (0.3.4)? Now that the linker knows
every real address, it goes back and fills them in. This is called **applying
relocations**.

```
   Before (in main.o, from the assembler):
        call <blank>           ← "call add, but I don't know its address"
        sticky note: "fill this with the address of `add`"

   The linker now knows: add is at 0x1150. So it fills the blank:
        call 0x1150            ← done!
```

In our book analogy: the typeset chapter said "see the recipe on page ???", and
now the bindery, having numbered all the pages, goes back and writes "see the
recipe on page 500" into every such gap.

```
   ┌───────────────────────────────────────────────────────────────┐
   │  THE WHOLE LINKER IN ONE SENTENCE:                            │
   │  Stack the pieces, number the pages (LAYOUT),                 │
   │  match every "need" to a "have" (RESOLUTION),                 │
   │  and write the real page numbers into all the blanks (RELOC). │
   └───────────────────────────────────────────────────────────────┘
```

If you understand that sentence, you understand linking.

---

## 0.4.6 Libraries: static vs dynamic (the beginner version)

We met this in 0.2.5; now let's make it concrete, because it explains a *lot* of
real-world behavior.

### Static linking — copy the library in

The linker copies the needed library code directly into your program. Your
program becomes **self-contained**: it carries everything it needs.

```
   your code + a COPY of printf's code  =  one big self-sufficient file
   ✓ runs anywhere, nothing else needed
   ✗ bigger file; if the library is fixed for a bug, you must rebuild
```

### Dynamic linking — borrow the library at run time

The linker does *not* copy the library in. Instead it records a note: "this
program needs `libc` (the C library)." The actual connection happens **later,
each time you run the program**, performed by the **loader** (next chapter).

```
   your code + a NOTE "needs libc"  =  small file
   ✓ small; many programs share ONE copy of libc in memory
   ✓ fix a bug in libc once, every program benefits
   ✗ if libc is missing/incompatible at run time → it won't start
       ("cannot open shared object file", "version GLIBC_x.y not found")
```

```
   STATIC                              DYNAMIC
   ──────                              ───────
   [ you + printf copy ]              [ you ]───needs──▶ [ libc, shared ]
   complete, larger, standalone       small; printf is connected at run time
```

**Most programs are dynamic by default.** That's why a tiny "hello world" file
is only a few kilobytes even though `printf` is complicated — `printf` isn't
inside it; it's borrowed at run time. This also explains why you sometimes get
errors only when *running* a program (not when building it): the borrowing
happens at run time, so a missing library shows up then.

---

## 0.4.7 A complete worked picture

Let's trace a two-file program all the way through, tying everything together.

```
   FILES YOU WROTE:
     main.c :  calls add(2,3) and printf
     math.c :  defines add()

   STEP 1 — compile + assemble each file separately:
     main.c → main.o   { has main; NEEDS add; NEEDS printf;  blanks for both }
     math.c → math.o   { HAS add; needs nothing extra }

   STEP 2 — linker LAYOUT (stack + number pages):
     .text =  [ main's code ][ add's code ]
     addresses assigned:  main=0x1130,  add=0x1150

   STEP 3 — linker RESOLUTION (match needs to haves):
     main needs add    → found in math.o (0x1150)        ✓
     main needs printf → found in the C library          ✓ (dynamic: recorded as a note)

   STEP 4 — linker fills the BLANKS:
     main's "call <blank>"  becomes  "call 0x1150"  (add)
     main's "call printf"   wired to the library connection mechanism

   RESULT: one finished program file, ready to run.
```

Every single step here is something you now understand in plain English.

---

## 0.4.8 See linking with your own eyes (optional)

**Try it ▸** Watch resolution succeed, and watch it fail:

```bash
# Two files: one defines add, one uses it
cat > math.c <<'EOF'
int add(int a, int b){ return a + b; }
EOF
cat > main.c <<'EOF'
#include <stdio.h>
int add(int a, int b);                       /* "I promise add exists somewhere" */
int main(void){ printf("%d\n", add(2,3)); return 0; }
EOF

# Compile each to an object file (assembler stage), but DON'T link yet:
gcc -c math.c -o math.o
gcc -c main.c -o main.o

# Now LINK them together → success:
gcc main.o math.o -o prog
./prog                # prints 5

# Now try linking WITHOUT math.o → the famous error:
gcc main.o -o broken  # error: undefined reference to `add'
```

That last command produces `undefined reference to 'add'` — because `main.o`
needs `add`, but you didn't give the linker the file that defines it. You now
know *exactly* why.

**Try it ▸** See whether a program is static or dynamic, and what it borrows:

```bash
gcc main.c math.c -o prog        # default = dynamic
file prog                        # mentions "dynamically linked"
ldd prog                         # lists the libraries it borrows at run time (libc...)

gcc -static main.c math.c -o prog_static   # force static
file prog_static                 # "statically linked"
ls -l prog prog_static           # the static one is much bigger!
```

Seeing the size difference and the `ldd` list makes static-vs-dynamic real.

---

## 0.4.9 What you now understand

- The **linker** exists to combine separately-built pieces (your files +
  libraries) into one program and connect every reference to its definition.
- It has two jobs: **layout** (stack the pieces and assign real addresses) and
  **resolution** (match every undefined "need" to a defined "have").
- After layout, it **fills in the blanks** (relocations) the assembler left,
  now that real addresses are known.
- `undefined reference` = a need with no matching definition; `multiple
  definition` = the same name defined twice. You can now diagnose both.
- Libraries are linked **statically** (copied in, self-contained, bigger) or
  **dynamically** (borrowed at run time, smaller, shared) — and dynamic is the
  common default, which is why some failures only appear when you *run* a program.

You've now mastered the central idea of the entire guide in plain English. The
detailed [Part 5 (Linking)](../05-linking/01-static-linking.md) will show you the
exact mechanics, but the *story* is already yours.

Next, a gentle look at the file formats that hold all of this.

→ [0.5 — What are ELF and DWARF?](05-what-are-elf-and-dwarf.md)
