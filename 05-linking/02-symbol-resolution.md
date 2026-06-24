# 5.2 — Symbol Resolution, Archives & the ODR

Symbol resolution is where the linker matches every reference to exactly one
definition. It's also the source of the two most common (and most confusing)
build errors. This chapter covers the resolution algorithm, static archives,
weak symbols, link order, and the C++ One Definition Rule at the binary level.

---

## 5.2.1 The resolution problem

Each object contributes:
- **Defined** global/weak symbols (it provides these).
- **Undefined** (UND) references (it needs these from elsewhere).

The linker must, for every undefined reference, find a definition — and ensure
each global is defined **exactly once**.

```
   a.o          b.o          c.o
   ───          ───          ───
   def main     def helper   def util
   und helper   und util     und (none)
   und printf   und printf

   resolution: helper→b.o, util→c.o, printf→libc
               main→(it's the entry, defined in a.o)
   leftover UND: none → success
```

```
   The two failure modes:
   ┌────────────────────────────────┬──────────────────────────────────┐
   │ A reference with NO definition │ "undefined reference to `foo'"   │
   │ A symbol with TWO definitions  │ "multiple definition of `foo'"   │
   └────────────────────────────────┴──────────────────────────────────┘
```

---

## 5.2.2 The resolution rules (strong, weak, common)

The linker classifies each global definition:

```
   STRONG  : a normal defined global (function body, initialized variable)
   WEAK    : a .weak symbol (overridable)
   COMMON  : a tentative definition (uninitialized C global, -fcommon)
```

Resolution rules (the classic "Linkers & Loaders" rules):

```
   Rule 1: Multiple STRONG defs of the same name → ERROR (multiple definition).
   Rule 2: One STRONG + any number of WEAK → the STRONG wins (silently).
   Rule 3: Multiple WEAK, no strong → linker picks one (typically the first,
           or the largest for COMMON). No error.
   Rule 4: COMMON symbols of the same name → merged into one (largest size wins).
   Rule 5: A reference matched only by a WEAK that's absent → resolves to 0/NULL.
```

```
   strong vs weak example:
     a.c:  int x = 1;                  → STRONG x
     b.c:  __attribute__((weak)) int x;→ WEAK x
     link: x resolves to a.c's (strong wins). No error.

   the silent-bug version:
     a.c:  int x = 1;       (strong, int)
     b.c:  double x;        (-fcommon: common, 8 bytes)
     link (legacy -fcommon): merged! writing x in b corrupts neighbors.
            ← exactly why -fno-common is now the default.
```

> **C++ ▸** `inline` functions, template instantiations, and `inline`
> variables are emitted as **weak** definitions (in COMDAT groups, §7.3). That's
> rule 2/3 at work: every TU that uses `std::vector<int>::push_back` emits its
> own weak copy, and the linker keeps one, discarding the rest. No ODR error
> because they're weak + COMDAT-deduplicated.

---

## 5.2.3 Static archives (`.a`) and the pull-in model

A static library is just an `ar` archive: a bundle of `.o` files plus an index.
Crucially, the linker treats archives **differently** from plain objects:

```
   Plain .o on the command line:  ALWAYS linked in (all its code included).
   .o inside an archive:          included ONLY IF it resolves a currently
                                  pending undefined symbol.
```

```
   The archive scan (single left-to-right pass):

   pending UND set = { ... }
   when the linker reaches libfoo.a:
       repeat:
         for each member .o in the archive:
            if it DEFINES a symbol currently in the pending UND set:
                pull in that member (add its code, its own UNDs to the set)
       until a full pass adds nothing
```

```
   ┌───────────────────────────────────────────────────────────────┐
   │ libfoo.a = { math.o (def sqrt), io.o (def read), net.o (...) }│
   │ program references only sqrt →  ONLY math.o is pulled in;     │
   │ io.o and net.o are LEFT OUT of the final binary.              │
   └───────────────────────────────────────────────────────────────┘
```

This selective inclusion is why static linking can produce small binaries
(unused archive members are dropped) — and why **link order matters**.

---

## 5.2.4 Link order: the classic footgun

Because the archive scan is a single left-to-right pass, **a library must appear
*after* the objects that use it.**

```
   gcc main.o -lm           ✓  main.o needs sqrt; -lm comes after → resolved
   gcc -lm main.o           ✗  when -lm is scanned, no pending UND for sqrt yet
                               (main.o not seen). sqrt never pulled.
                               → "undefined reference to `sqrt'"
```

```
   Timeline (wrong order  gcc -lm main.o):
     scan -lm   : pending UND = {} → nothing in libm matches → skip everything
     scan main.o: now sqrt is UND, but libm already passed → ERROR
```

Circular dependencies between archives need either repetition or a group:

```
   gcc main.o -la -lb -la           # repeat -la
   gcc main.o -Wl,--start-group -la -lb -Wl,--end-group   # rescan until stable
```

> **Why not just scan everything multiple times?** Historically, for speed and
> determinism. Modern linkers are smarter but preserve the order-sensitive
> semantics for compatibility. `--start-group` forces repeated rescanning.

**Try it ▸** Reproduce the footgun:

```bash
printf '#include <math.h>\nint main(){return (int)sqrt(2.0);}\n' > m.c
gcc -c m.c
gcc -lm m.o -o bad   2>&1 || echo "FAILED (wrong order)"
gcc m.o -lm -o good  && echo "OK (correct order)"
```

---

## 5.2.5 `--gc-sections`: dropping dead code

Even within objects you *do* link, unused functions can be discarded if each
lives in its own section:

```
   compile:  -ffunction-sections -fdata-sections   (one section per symbol)
   link:     -Wl,--gc-sections                     (garbage-collect unreached)

   The linker starts from "roots" (entry point, exported symbols, KEEP'd
   sections) and marks everything reachable via relocations; unmarked sections
   are deleted.
```

```
   root: _start ──▶ main ──▶ foo ──▶ bar
                                  baz (never referenced) ──▶ DROPPED
```

This is reachability GC over the section/relocation graph. `-Wl,--print-gc-sections`
shows what was removed. Embedded and size-sensitive builds rely on it heavily.

---

## 5.2.6 Symbol interposition and `LD_PRELOAD`

In **dynamic** linking, the *first* definition found across all loaded modules
wins — this is **interposition**, and it's how `LD_PRELOAD` overrides functions:

```
   search order: executable, then each DT_NEEDED library in order, then deps...
   LD_PRELOAD libs are inserted at the FRONT → they win over libc.

   LD_PRELOAD=./mymalloc.so ./app
   → app's calls to malloc bind to mymalloc.so's malloc, not libc's.
```

```
   default symbol lookup scope (global):
   [ LD_PRELOAD libs ] → [ executable ] → [ lib1 ] → [ lib2 ] → ...
        first match wins (for STV_DEFAULT visibility symbols)
```

Interposition is powerful (sanitizers, profilers, mocking) but has a cost: the
compiler can't assume a `STV_DEFAULT` global function in a shared library won't
be replaced, so it routes calls through the PLT (§5.4). `-fno-semantic-interposition`
or `STV_PROTECTED`/`STV_HIDDEN` visibility lets it optimize.

---

## 5.2.7 The C++ One Definition Rule at the binary level

The ODR says: each entity has exactly one definition across the program (with
careful exceptions for inline entities that must be *identical* in every TU). The
linker enforces a *mechanical shadow* of this:

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │ Entity kind            │ Emitted as       │ Duplicate handling       │
   ├────────────────────────┼──────────────────┼──────────────────────────┤
   │ normal function/global │ STRONG           │ duplicate → link error   │
   │ inline function        │ WEAK + COMDAT    │ keep one, discard rest   │
   │ template instantiation │ WEAK + COMDAT    │ keep one, discard rest   │
   │ inline variable (C++17)│ WEAK + COMDAT    │ keep one                 │
   │ vtable / typeinfo      │ WEAK + COMDAT    │ keep one (per key method)│
   │ static local in inline │ STB_GNU_UNIQUE   │ one shared instance      │
   └────────────────────────┴──────────────────┴──────────────────────────┘
```

```
   COMDAT dedup:  every TU that instantiates std::vector<int> emits
                  _ZNSt6vectorIiE...  as a weak COMDAT group; the linker
                  keeps ONE and drops the duplicates → no "multiple definition".
```

> **The dangerous ODR violation ▸** If two TUs provide *different* definitions
> of the same inline/template entity (e.g. compiled with different `-DNDEBUG`,
> different struct layouts, or different optimization that changes inline bodies
> in incompatible ways), the linker silently keeps **one arbitrary** copy. No
> diagnostic, and the program may behave inconsistently. This is undefined
> behavior and one of the nastiest C++ bugs — the linker's COMDAT mechanism
> assumes all copies are identical and doesn't verify it. Tools: `gold`'s
> `--detect-odr-violations`, the `-flto` whole-program view, and link-time ODR
> checkers.

---

## 5.2.8 Diagnosing resolution problems

```bash
# WHO is undefined / multiply defined?
nm -C app.o | grep ' U '          # undefined symbols (capital U)
nm -C lib.a                        # what an archive provides

# WHICH object would satisfy a symbol?
nm -CA *.o | grep ' T main'        # -A prefixes the filename

# WHY was an archive member pulled in? (lld)
ld.lld --why-extract=- ... 

# Trace symbol resolution
gcc ... -Wl,--trace-symbol=foo     # report every reference/definition of foo
gcc ... -Wl,-y,foo                 # same (short form)
```

> **Reading the error ▸** `undefined reference to 'foo'` lists *where it was
> referenced* (the call site object). To fix: ensure something *defines* `foo`
> and that the defining object/library comes **after** the user on the command
> line. `multiple definition` lists both definition sites; remove one or mark it
> `inline`/`weak`/`static`.

---

## Summary

- Resolution matches each undefined reference to exactly one definition;
  failures are "undefined reference" (none) or "multiple definition" (two
  strong).
- Strong/weak/common rules decide ties: strong beats weak, commons merge; this
  is the machinery behind `inline`/template dedup and the `-fno-common` story.
- Static archives are scanned **left-to-right, on demand** — only members that
  resolve pending undefineds are pulled in, which makes **link order**
  significant (libraries after their users; groups for cycles).
- `--gc-sections` GC-collects unreachable per-symbol sections.
- Dynamic linking adds **interposition** (`LD_PRELOAD`, first definition wins),
  which constrains optimization.
- The linker mechanically enforces a shadow of the C++ **ODR** via WEAK+COMDAT;
  *inconsistent* duplicate definitions are a silent, dangerous UB.

Next: [5.3 — Dynamic linking & shared objects](03-dynamic-linking.md)
