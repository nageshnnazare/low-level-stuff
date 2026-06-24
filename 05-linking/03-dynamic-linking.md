# 5.3 — Dynamic Linking & Shared Objects

Dynamic linking defers part of the link to *load time*, letting many programs
share one copy of a library in memory and on disk, and letting libraries be
updated independently. This chapter covers shared objects, the `.dynamic`
section, dependency resolution, and the search algorithm.

---

## 5.3.1 Static vs dynamic linking: the trade

```
 ┌──────────────────────┬──────────────────────────┬──────────────────────────┐
 │                      │ STATIC                   │ DYNAMIC                  │
 ├──────────────────────┼──────────────────────────┼──────────────────────────┤
 │ Library code lives   │ copied INTO each exe     │ in a shared .so, once    │
 │ Disk usage           │ duplicated per program   │ one copy shared          │
 │ RAM usage            │ duplicated per process   │ .text pages shared (COW) │
 │ Update a library     │ relink every program     │ replace the .so          │
 │ Startup cost         │ ~none                    │ ld.so maps + relocates   │
 │ Runtime call cost    │ direct                   │ PLT/GOT indirection      │
 │ Distribution         │ self-contained binary    │ needs the .so present    │
 │ Security fixes       │ rebuild everything       │ patch one .so            │
 └──────────────────────┴──────────────────────────┴──────────────────────────┘
```

```
   STATIC:   [ app+libc copy ]  [ app2+libc copy ]   ← libc duplicated
   DYNAMIC:  [ app ]──┐  [ app2 ]──┐
                      └──▶ libc.so ◀──┘               ← one shared libc
```

Most Linux software is dynamically linked. Fully static binaries (`-static`) are
making a comeback for containers/portability (e.g. Go, musl-based images), at the
cost of size and the loss of `dlopen`/NSS plugins.

---

## 5.3.2 What makes an ELF "dynamic"

Two things turn a file into a participant in dynamic linking:

```
   1. A PT_INTERP segment naming the dynamic loader
      (executables only; says "run me via /lib64/ld-linux-x86-64.so.2").
   2. A PT_DYNAMIC segment / .dynamic section
      (both executables and shared libs; the control block for ld.so).
```

A shared library (`.so`, `ET_DYN`) has `.dynamic` but **no** `PT_INTERP` (it is
not directly executed). A dynamically-linked executable has both.

**Try it ▸**

```bash
readelf -d demo | head        # the .dynamic array
readelf -p .interp demo       # the interpreter path
file /lib/x86_64-linux-gnu/libc.so.6   # ET_DYN, no interp needed
```

---

## 5.3.3 The `.dynamic` array (`Elf64_Dyn`)

`.dynamic` is the loader's table of contents — an array of `(tag, value)` pairs
terminated by `DT_NULL`:

```c
typedef struct {
    Elf64_Sxword d_tag;
    union { Elf64_Xword d_val; Elf64_Addr d_ptr; } d_un;
} Elf64_Dyn;
```

The essential tags `ld.so` reads:

```
 ┌──────────────┬─────────────────────────────────────────────────────────────┐
 │ DT_NEEDED    │ name (in .dynstr) of a required shared library. One per dep.│
 │ DT_SONAME    │ this library's canonical name (e.g. libfoo.so.1)            │
 │ DT_RPATH /   │ extra library search paths baked into the binary            │
 │   DT_RUNPATH │   (RUNPATH is the modern, scoped form)                      │
 │ DT_STRTAB    │ address of .dynstr                                          │
 │ DT_SYMTAB    │ address of .dynsym                                          │
 │ DT_HASH /    │ symbol hash table (fast lookup)                             │
 │   DT_GNU_HASH│                                                             │
 │ DT_RELA      │ address of .rela.dyn ; DT_RELASZ size ; DT_RELAENT stride   │
 │ DT_JMPREL    │ address of .rela.plt ; DT_PLTRELSZ size ; DT_PLTREL kind    │
 │ DT_PLTGOT    │ address of the GOT (specifically .got.plt)                  │
 │ DT_INIT /    │ legacy single init/fini function (_init/_fini)              │
 │   DT_FINI    │                                                             │
 │ DT_INIT_ARRAY│ array of ctor pointers ; DT_INIT_ARRAYSZ                    │
 │ DT_FINI_ARRAY│ array of dtor pointers ; DT_FINI_ARRAYSZ                    │
 │ DT_FLAGS /   │ behavior flags: BIND_NOW, NODELETE, ...                     │
 │   DT_FLAGS_1 │   DF_1_PIE, DF_1_NOW, DF_1_NODELETE, ...                    │
 │ DT_RELR      │ compressed relative relocations (modern)                    │
 │ DT_NULL      │ end marker                                                  │
 └──────────────┴─────────────────────────────────────────────────────────────┘
```

```
   So ld.so's entire job is driven by .dynamic:
     1. read DT_NEEDED list → load those libraries (recursively)
     2. use DT_SYMTAB/DT_STRTAB/DT_GNU_HASH to resolve symbols
     3. apply DT_RELA/DT_JMPREL/DT_RELR relocations
     4. run DT_INIT_ARRAY constructors
```

**Try it ▸**

```bash
readelf -d demo
# Look for NEEDED (libc.so.6), the *RELA*/*JMPREL* tags, INIT_ARRAY, FLAGS_1.
```

---

## 5.3.4 SONAME, real names, and the version dance

Shared libraries use a three-name scheme to allow safe upgrades:

```
   linker name :  libfoo.so          (a symlink; used at LINK time with -lfoo)
   soname      :  libfoo.so.1        (the ABI-compatible major version; baked
                                      into dependents as DT_NEEDED)
   real name   :  libfoo.so.1.4.2    (the actual file; full version)

   filesystem:
     libfoo.so        → libfoo.so.1       (dev symlink, from -dev package)
     libfoo.so.1      → libfoo.so.1.4.2   (runtime symlink, ldconfig-managed)
     libfoo.so.1.4.2  = the real ELF, DT_SONAME = "libfoo.so.1"
```

```
   At LINK time:   -lfoo finds libfoo.so, reads its DT_SONAME ("libfoo.so.1"),
                   and records DT_NEEDED = libfoo.so.1 in your binary.
   At LOAD time:   ld.so looks for libfoo.so.1 (the soname), following the
                   runtime symlink to whatever 1.x.y is installed.
```

The contract: bumping the **soname major** (`.so.1` → `.so.2`) signals an ABI
break; minor/patch upgrades keep the same soname so old binaries keep working.
This is semantic versioning enforced by the loader. `ldconfig` maintains the
symlinks and a cache (`/etc/ld.so.cache`).

---

## 5.3.5 The library search algorithm

When `ld.so` needs `libfoo.so.1` (a `DT_NEEDED`), it searches in this order:

```
   1. DT_RPATH of the loading object        (deprecated; searched before LD_LIBRARY_PATH)
   2. LD_LIBRARY_PATH environment variable  (skipped for setuid/secure programs)
   3. DT_RUNPATH of the loading object      (searched AFTER LD_LIBRARY_PATH)
   4. /etc/ld.so.cache                       (the ldconfig cache; the fast path)
   5. default dirs: /lib, /usr/lib, /lib64, /usr/lib64 (+ multiarch dirs)
```

```
   ┌─────────────┐   not found   ┌──────────────────┐   not found  ┌───────────┐
   │ DT_RPATH    │ ─────────────▶│ LD_LIBRARY_PATH  │ ────────────▶│ DT_RUNPATH│
   └─────────────┘               └──────────────────┘              └─────┬─────┘
                                                                         │
        ┌────────────────────────────────────────────────────────────────┘
        ▼                          not found
   ┌──────────────────┐  ───────────────────▶   ┌──────────────────────────┐
   │ /etc/ld.so.cache │                         │ /lib /usr/lib /lib64 ... │
   └──────────────────┘                         └──────────────────────────┘
                                                       │ still not found
                                                       ▼
                                       "error while loading shared libraries:
                                        libfoo.so.1: cannot open shared object file"
```

> **RPATH vs RUNPATH ▸** Both bake search paths into the binary, but **RPATH**
> is checked *before* `LD_LIBRARY_PATH` (so it can't be overridden — historically
> a security issue), while **RUNPATH** is checked *after* (overridable). Modern
> linkers emit RUNPATH (`-Wl,--enable-new-dtags`). Use `$ORIGIN` for relocatable
> installs: `-Wl,-rpath,'$ORIGIN/../lib'` makes a binary find its libs relative
> to its own location.

**Try it ▸**

```bash
ldd demo                                  # resolved dependency paths
readelf -d demo | grep -E 'NEEDED|RUNPATH|RPATH|SONAME'
LD_DEBUG=libs ./demo 2>&1 | head -20      # trace the search live
ldconfig -p | grep libc.so                # what the cache knows
```

---

## 5.3.6 Explicit dynamic loading: `dlopen`

Beyond automatic `DT_NEEDED` loading, programs can load libraries at runtime:

```c
#include <dlfcn.h>
void *h = dlopen("libplugin.so", RTLD_NOW | RTLD_LOCAL);
auto fn = (int(*)(int)) dlsym(h, "plugin_entry");   // by name
int r = fn(42);
dlclose(h);
```

```
   RTLD_LAZY  : resolve symbols on first use (lazy)
   RTLD_NOW   : resolve all symbols immediately at dlopen
   RTLD_GLOBAL: this lib's symbols become available to subsequently loaded libs
   RTLD_LOCAL : keep its symbols private (default)
```

This is how plugin systems work. Note `dlsym` needs the *mangled* name for C++
symbols (or `extern "C"` entry points). `dlopen` is also why fully-static
binaries are limited — you can't `dlopen` into a truly static image easily, and
glibc's NSS/iconv plugins rely on it.

> **C++ ▸** Throwing exceptions across a `dlopen`ed boundary, RTTI/`dynamic_cast`
> across modules, and duplicate vtables/type_info require care: type identity is
> by `type_info` *address*, so the same type in two independently-loaded modules
> can compare unequal unless symbols are shared (STV_DEFAULT + RTLD_GLOBAL, or a
> common base library). STB_GNU_UNIQUE and `--dynamic-list-cpp-typeinfo` exist to
> address this.

---

## 5.3.7 Building and using a shared library (full demo)

**Try it ▸**

```bash
# 1. Build a shared library (PIC required)
cat > mylib.c <<'EOF'
int square(int x){ return x*x; }
EOF
gcc -fPIC -c mylib.c -o mylib.o
gcc -shared -Wl,-soname,libmylib.so.1 mylib.o -o libmylib.so.1.0.0
ln -sf libmylib.so.1.0.0 libmylib.so.1
ln -sf libmylib.so.1     libmylib.so

# 2. Link a program against it
cat > useit.c <<'EOF'
#include <stdio.h>
int square(int);
int main(void){ printf("%d\n", square(7)); return 0; }
EOF
gcc useit.c -L. -lmylib -Wl,-rpath,'$ORIGIN' -o useit

# 3. Inspect & run
readelf -d useit | grep -E 'NEEDED|SONAME|RUNPATH'   # NEEDED libmylib.so.1
./useit                                              # 49
```

Notice the `DT_NEEDED` records the **soname** (`libmylib.so.1`), not the linker
name, exactly as §5.3.4 describes.

---

## Summary

- Dynamic linking shares library code across programs/processes and decouples
  upgrades, at the cost of load-time work and PLT/GOT call indirection.
- A file participates via `PT_INTERP` (executables) and `PT_DYNAMIC`/`.dynamic`
  (both); `.dynamic` is the loader's table of contents (`DT_NEEDED`, `DT_RELA`,
  `DT_SYMTAB`, `DT_INIT_ARRAY`, …).
- The three-name scheme (linker name / **soname** / real name) plus `ldconfig`
  symlinks lets libraries upgrade without breaking dependents; the soname major
  is the ABI promise.
- `ld.so` finds libraries via RPATH → `LD_LIBRARY_PATH` → RUNPATH → ld.so.cache
  → default dirs; prefer RUNPATH and `$ORIGIN`.
- `dlopen`/`dlsym` provide explicit runtime loading for plugins; C++ type
  identity across modules needs attention.

Next: [5.4 — PLT, GOT & lazy binding](04-plt-got.md)
