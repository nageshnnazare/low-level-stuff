# 5.1 — Static Linking From Start to Finish

The linker is where everything from Parts 3 and 4 converges. This chapter walks
the full static-linking pipeline: collecting sections, laying out the address
space, resolving symbols, and applying relocations to produce a runnable file.

---

## 5.1.1 The linker's mandate

> Combine one or more relocatable objects (and libraries) into a single output
> file: merge their sections, assign final addresses, resolve every symbol
> reference to a definition, apply all relocations, and emit the result
> (executable or shared object).

```
   INPUTS                          LINKER                       OUTPUT
   ──────                          ──────                       ──────
   a.o  ─┐                  ┌──────────────────────┐
   b.o  ─┼─────────────────▶│ 1 collect sections   │
   c.o  ─┤                  │ 2 resolve symbols    │──────────▶ a.out
   libm.a (archive) ────────│ 3 lay out memory     │           (ET_EXEC/
   libc.so (shared) ────────│ 4 apply relocations  │            ET_DYN)
                            │ 5 build metadata     │
                            │ 6 write the ELF      │
                            └──────────────────────┘
```

The same tool (`ld`) produces both executables and shared libraries; flags and
inputs differ. "Static linking" here means the *link step itself*; whether libc
is bundled in (fully static) or referenced (dynamic) is a separate axis (§5.3).

---

## 5.1.2 Phase 1 — Section collection and merging

The linker reads each input object's section table and groups sections by
**output section** (usually by name, governed by a *linker script*):

```
   a.o:  .text(0x40) .data(0x10) .rodata(0x8)
   b.o:  .text(0x30) .data(0x08) .bss(0x100)
   c.o:  .text(0x20)             .bss(0x040)

   ──merge──▶  output .text   = a.text ++ b.text ++ c.text   (0x90 bytes)
               output .data   = a.data ++ b.data             (0x18 bytes)
               output .rodata = a.rodata                     (0x08 bytes)
               output .bss    = b.bss ++ c.bss               (0x140 bytes)
```

```
   Concatenation keeps track of each input's OFFSET within the output section:

   output .text:
   ┌───────────── a.text ─────────────┬──── b.text ────┬── c.text ──┐
   0x00                              0x40            0x70          0x90
        a's symbols keep offset      b's symbols     c's symbols
        relative to 0x00             rebased to 0x40 rebased to 0x70
```

Each input section's contribution gets a base offset; the linker records this so
it can later translate `(input section, offset)` symbol values into final
addresses. With `-ffunction-sections`, there are many tiny `.text.*` inputs that
merge into one `.text` (and unused ones are dropped by `--gc-sections`).

---

## 5.1.3 Phase 2 — Symbol resolution (overview)

The linker builds a global symbol table, matching each **undefined** reference to
a **definition** by name. (Full detail — archives, weak symbols, ODR — is §5.2.)

```
   global symbol table being built:
   ┌────────────┬───────────┬─────────────────────────────┐
   │ name       │ state     │ provider                    │
   ├────────────┼───────────┼─────────────────────────────┤
   │ main       │ DEFINED   │ a.o .text+0x00              │
   │ helper     │ DEFINED   │ b.o .text+0x00 (→ out 0x40) │
   │ printf     │ UNDEFINED │ → pulled from libc          │
   │ g_counter  │ DEFINED   │ b.o .data+0x00              │
   └────────────┴───────────┴─────────────────────────────┘

   If any required symbol stays UNDEFINED → "undefined reference" error.
   If two objects DEFINE the same global → "multiple definition" error.
```

---

## 5.1.4 Phase 3 — Memory layout (address assignment)

Now the linker assigns **virtual addresses**. It orders output sections, packs
them into segments, and respects alignment and the default linker script.

```
   For a non-PIE ET_EXEC (classic), the default base is 0x400000:

   0x400000  ┌─────────────────────┐  ← image base
             │ ELF hdr + phdrs     │
   0x401000  ├─────────────────────┤  ← R-X segment (page aligned)
             │ .init .plt .text    │
             │ .fini               │
   0x402000  ├─────────────────────┤  ← R-- segment
             │ .rodata .eh_frame   │
   0x403000  ├─────────────────────┤  ← RW- segment
             │ .data               │
             │ .bss (no file bytes)│
             └─────────────────────┘

   For a PIE (ET_DYN), addresses start near 0 (e.g. 0x1000) and the loader
   adds a randomized base at runtime.
```

During layout the linker:
- Assigns each output section an address (honoring `sh_addralign`, page
  alignment for segment boundaries).
- Resolves the value of every defined symbol to a concrete virtual address
  (`base_of_section + offset_within_section`).
- Decides the entry point (`_start`).
- Computes the program header table (segments).

> **Linker scripts ▸** The layout policy lives in a *linker script* (default
> built into `ld`; see `ld --verbose`). It maps input section patterns to output
> sections and assigns addresses. Embedded/OS developers write custom scripts
> (`-T script.ld`) to place code at specific addresses (e.g. a kernel at
> `0xffffffff80000000`, or firmware in flash). The `.` symbol in a linker script
> is the location counter, just like in the assembler.

---

## 5.1.5 Phase 4 — Relocation application

With all addresses known, the linker walks every relocation (Part 4.6) and
patches the bytes. References that can be fully resolved now (static relocations)
are computed and the records discarded. References that still need runtime help
(dynamic relocations) are converted into `.rela.dyn`/`.rela.plt` + GOT/PLT
entries (§5.4).

```
   for each input relocation R with target T:
       S = final address of R's symbol
       P = final address of the field (T's base + r_offset)
       A = r_addend
       write f(type, S, A, P, ...) into the output bytes at P
       (or, if it needs runtime resolution, emit a dynamic reloc + GOT slot)
```

This is the same five-step computation from §4.6.3, performed for every
reference in the program.

---

## 5.1.6 Phase 5 — Generating metadata and output

Finally the linker synthesizes:
- The **program header table** (segments) for the loader.
- For dynamic output: `.dynamic`, `.dynsym`/`.dynstr`, `.gnu.hash`, `.plt`,
  `.got`/`.got.plt`, version sections, and the `PT_INTERP` naming `ld.so`.
- `.init_array`/`.fini_array` ordering, `_init`/`_fini`.
- `.eh_frame_hdr` (a sorted index over `.eh_frame` for fast unwinding).
- Optionally a build-id note, and a (kept or stripped) symbol table.

Then it writes the ELF file.

---

## 5.1.7 The CRT objects: what the driver secretly adds

When you run `gcc main.o -o app`, the driver doesn't link *just* your object. It
prepends/appends the **C runtime startup** objects and default libraries:

```
   Effective link command (simplified):

   ld -dynamic-linker /lib64/ld-linux-x86-64.so.2          \
      /usr/lib/.../Scrt1.o  crti.o  crtbeginS.o            \  ← startup (before)
      main.o                                               \  ← YOUR code
      -lstdc++ -lm -lgcc_s -lc                             \  ← default libs
      crtendS.o  crtn.o                                    \  ← shutdown (after)
      -o app
```

| Object        | Role                                                       |
|---------------|------------------------------------------------------------|
| `Scrt1.o`/`crt1.o` | Defines `_start`; calls `__libc_start_main(main,...)` |
| `crti.o`      | Function *prologues* for `_init`/`_fini` (frame open)      |
| `crtbeginS.o` | Per-TU ctor/dtor registration scaffolding (GCC)            |
| `crtendS.o`   | …matching epilogue scaffolding                             |
| `crtn.o`      | Function *epilogues* for `_init`/`_fini` (frame close)     |

```
   _init() body = crti.o's open  ++  (compiler-contributed)  ++  crtn.o's close
   This bracketing is why crti/crtn must surround the others in link order.
```

**Try it ▸** See the real command and the CRT objects:

```bash
gcc -v hello.c -o hello 2>&1 | grep -E 'collect2|crt1|crti|crtbegin'
```

---

## 5.1.8 Watching the linker work

**Try it ▸** Two objects, a map file, and the resulting layout:

```bash
cat > a.c <<'EOF'
#include <stdio.h>
int g = 5;
extern int helper(int);
int main(void){ printf("%d\n", helper(g)); return 0; }
EOF
cat > b.c <<'EOF'
int helper(int x){ return x*x; }
EOF
gcc -c a.c b.c
gcc a.o b.o -o app -Wl,-Map=app.map
# Inspect:
grep -A2 -E '\.text|\.data|\.bss' app.map | head -30   # where each input landed
nm -n app | grep -E ' main| helper| g$'                # final addresses, sorted
readelf -l app                                         # the segments produced
```

The map file shows, for each output section, every contributing input object and
its assigned address — the concatenation from §5.1.2 made concrete.

---

## 5.1.9 Modern linkers and why they differ

```
 ┌────────┬─────────────────────────────────────────────────────────────────┐
 │ ld.bfd │ The traditional GNU linker (binutils). Most features, slowest.  │
 │ gold   │ Faster ELF-only GNU linker (incremental, threads). Now legacy.  │
 │ lld    │ LLVM's linker. Very fast, multithreaded, drop-in for ld.        │
 │ mold   │ Newest, fastest; aggressive parallelism. -fuse-ld=mold.         │
 └────────┴─────────────────────────────────────────────────────────────────┘
```

They all implement the same five phases and the same ELF semantics; they differ
in speed, parallelism, LTO integration, and exotic features. Select one with
`-fuse-ld=lld` / `-fuse-ld=mold`. For huge C++ codebases, `lld`/`mold` cut link
times from minutes to seconds — link time often dominates incremental C++ builds.

---

## Summary

- Static linking = collect/merge sections → resolve symbols → assign addresses →
  apply relocations → emit metadata → write the ELF.
- Same-named input sections are concatenated; each input remembers its offset so
  symbol values become final addresses.
- Address assignment follows a linker script (default or custom); the `.`
  location counter and section→address mapping live there.
- Relocation application is the §4.6.3 computation done program-wide; what can't
  resolve now becomes a dynamic relocation + GOT/PLT.
- The `gcc` driver silently adds CRT objects (`Scrt1/crti/crtbegin/...`) and
  default libraries; `_start` (not `main`) is the entry point.
- All modern linkers (bfd/gold/lld/mold) share semantics but differ wildly in
  speed.

Next: [5.2 — Symbol resolution, archives & the ODR](02-symbol-resolution.md)
