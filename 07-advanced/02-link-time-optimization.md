# 7.2 — Link-Time Optimization (LTO)

Traditional compilation optimizes each translation unit in isolation; the linker
just stitches the results. **LTO** breaks this barrier by deferring optimization
until link time, when the *whole program* is visible — enabling cross-TU
inlining, dead-code elimination, and devirtualization. This chapter explains how
it reshapes the toolchain.

---

## 7.2.1 The optimization barrier of separate compilation

```
   a.cpp ─compile▶ a.o   (optimized knowing ONLY a.cpp)
   b.cpp ─compile▶ b.o   (optimized knowing ONLY b.cpp)
                    │
                    └─link▶ app   (linker can't re-optimize across the boundary)

   Problem: a() in a.cpp calls b() in b.cpp. The compiler couldn't inline b()
   into a() — it never saw b()'s body. Lost optimization.
```

Each `.o` is already lowered to machine code; the linker only relocates and
concatenates. Cross-module inlining, constant propagation across files, and
whole-program devirtualization are all impossible.

---

## 7.2.2 The LTO idea: ship IR, optimize at link

LTO changes what goes into the `.o`: instead of (or alongside) machine code, the
compiler emits its **intermediate representation** (GIMPLE for GCC, LLVM bitcode
for Clang). The linker, via a **plugin**, hands all the IR back to the compiler's
optimizer, which now sees everything at once.

```
   a.cpp ─▶ a.o  (contains GIMPLE/bitcode IR, not final machine code)
   b.cpp ─▶ b.o  (IR)
                  │
                  ▼
   link step: LTO plugin ─▶ compiler re-runs the optimizer on COMBINED IR
                            (inline b() into a(), DCE, devirtualize, ...)
                          ─▶ THEN generates machine code ─▶ relocate ─▶ app
```

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  Normal:   parse → optimize(per-TU) → codegen │ link               │
   │  LTO:      parse → (emit IR)                  │ link: optimize     │
   │                                               │ (whole program)    │
   │                                               │ → codegen → emit   │
   └────────────────────────────────────────────────────────────────────┘
```

The linker isn't doing the optimization itself — it **calls back** into the
compiler through the **linker plugin** interface (`liblto_plugin.so` /
`LLVMgold.so`). The `.o` files contain "fat" (IR + machine code) or "slim" (IR
only) objects.

---

## 7.2.3 What LTO unlocks

```
   Cross-module INLINING:     inline small hot functions across .cpp boundaries
   Whole-program DCE:         remove functions/globals unused by the ENTIRE program
   Cross-module CONST-PROP:   propagate constants through call boundaries
   DEVIRTUALIZATION:          if the whole program shows only one override of a
                              virtual, replace the indirect call with a direct one
   IPA (interprocedural):     better alias analysis, attribute inference (pure/const,
                              noexcept), argument promotion across files
```

> **C++ ▸** LTO is especially powerful for C++ because heavy abstraction
> (templates, small accessor methods, virtual dispatch) creates many tiny
> cross-TU calls that *should* be free. Whole-program devirtualization can turn
> `shape->area()` into a direct call when the linker proves only one `area`
> override is ever linked in — something no per-TU pass can do safely.

---

## 7.2.4 Full vs Thin LTO

Monolithic LTO doesn't scale: merging all IR into one giant module and
optimizing serially is slow and memory-hungry for large programs. **ThinLTO**
(LLVM) makes it parallel and incremental:

```
 ┌──────────────────────┬────────────────────────────────────────────────────────┐
 │ Full / Monolithic LTO│ Merge ALL modules → one huge IR → optimize serially.   │
 │                      │ Best results, but slow & huge memory. Doesn't scale.   │
 ├──────────────────────┼────────────────────────────────────────────────────────┤
 │ ThinLTO              │ 1. cheap GLOBAL summary pass builds a call graph +     │
 │                      │    import/export summary across all modules.           │
 │                      │ 2. each module optimized IN PARALLEL, importing only   │
 │                      │    the few functions it needs from others.             │
 │                      │ Near-full-LTO quality, near-normal build times,        │
 │                      │ cacheable & incremental.                               │
 └──────────────────────┴────────────────────────────────────────────────────────┘
```

```
   ThinLTO pipeline:
     phase 1 (thin link):   read all module summaries → decide cross-module
                            import lists → write an index
     phase 2 (parallel):    [module a | module b | module c | ...]  optimized
                            concurrently, each importing its needed functions
```

ThinLTO is what makes LTO practical for Chromium-scale C++ (millions of lines).

---

## 7.2.5 Using LTO

**Try it ▸**

```bash
cat > a.cpp <<'EOF'
int helper(int x);                  // declaration only
int compute(int n){ return helper(n) + helper(n+1); }
EOF
cat > b.cpp <<'EOF'
int helper(int x){ return x*x; }    // small, should be inlined cross-TU
EOF
cat > main.cpp <<'EOF'
#include <cstdio>
int compute(int);
int main(){ printf("%d\n", compute(3)); }
EOF

# Without LTO: helper stays a separate call in compute()
g++ -O2 -c a.cpp b.cpp main.cpp
g++ -O2 a.o b.o main.o -o nolto
objdump -d nolto | sed -n '/<_Z7computei>:/,/ret/p'   # see a call to helper

# With LTO: helper is inlined into compute (no call)
g++ -O2 -flto -c a.cpp b.cpp main.cpp
g++ -O2 -flto a.o b.o main.o -o withlto
objdump -d withlto | sed -n '/<_Z7computei>:/,/ret/p'  # helper inlined away

# ThinLTO (clang):
clang++ -O2 -flto=thin -c a.cpp b.cpp main.cpp && clang++ -O2 -flto=thin *.o -o thin
```

Confirm the plugin is engaged:

```bash
g++ -flto -v ... 2>&1 | grep -i plugin     # liblto_plugin
ar p libfoo.a | file -                     # LTO objects show "LLVM IR bitcode" / GIMPLE
nm a.o                                      # LTO .o often shows few/no normal symbols
```

---

## 7.2.6 Practical considerations & pitfalls

```
   + Smaller, faster binaries (cross-module inlining + whole-program DCE).
   + Better devirtualization and IPA.

   − Longer, more memory-hungry link step (mitigated by ThinLTO).
   − Debugging is harder: inlining muddies backtraces (DWARF inlined_subroutine
     helps; use -g with LTO).
   − Build-system friction: LTO .o files aren't normal objects — `ar`/`nm`/
     mixing compilers/versions can break; use the gcc-ar/gcc-nm wrappers that
     load the plugin.
   − Mixing LTO and non-LTO objects: works (non-LTO objects just aren't
     re-optimized), but you lose cross-module benefits at those boundaries.
   − ODR violations become MORE dangerous: LTO's whole-program view can expose
     (or mis-merge) inconsistent inline definitions; it can also DETECT some.
```

> **Determinism & reproducible builds ▸** LTO (especially with parallelism)
> historically threatened build reproducibility; modern toolchains pin ordering
> for deterministic output. ThinLTO caches per-module results, so incremental
> rebuilds only re-optimize changed modules and their importers.

> **`-ffat-lto-objects` ▸** Emits both IR *and* machine code in each `.o`, so the
> object is usable by both LTO-aware and legacy linkers/tools, at the cost of
> bigger objects. "Slim" objects (IR only) are smaller but require the plugin
> everywhere.

---

## Summary

- Separate compilation creates an optimization barrier; **LTO** defers
  optimization to link time by shipping compiler **IR** (GIMPLE/LLVM bitcode) in
  the `.o` files.
- The linker calls back into the compiler via a **linker plugin** to optimize the
  whole program, then generate machine code — enabling cross-module inlining,
  whole-program DCE, cross-TU constant propagation, and devirtualization
  (especially valuable for abstraction-heavy C++).
- **Full LTO** gives the best results but doesn't scale; **ThinLTO** uses a cheap
  global summary + parallel per-module optimization for near-full quality at
  practical build times.
- Costs: slower/heavier links, muddier debugging, build-system friction (use
  `gcc-ar`/`gcc-nm`), and amplified sensitivity to ODR violations.

Next: [7.3 — C++ name mangling, vtables & COMDAT](03-cpp-abi-mangling.md)
