# Part 8 — Guided Labs & Exercises

Reading builds intuition; *doing* builds mastery. These labs are ordered to match
the guide and are designed to be done in a real ELF/Linux environment. Each has a
goal, steps, and "what you should observe." Do them in order; later labs assume
earlier tools.

> **Setup.** On Linux, install: `build-essential gdb binutils elfutils
> dwarfdump file clang lld radare2`. On macOS, run them inside Docker:
> `docker run --rm -it -v "$PWD":/work -w /work ubuntu:24.04 bash` then
> `apt-get update && apt-get install -y build-essential gdb binutils elfutils
> dwarfdump file clang lld radare2`. (macOS native uses Mach-O/dyld, not
> ELF/ld.so — the concepts transfer but the tools/output differ.)

---

## Lab 0 — Warm-up: the four stages

**Goal:** see each toolchain stage's output. (Part 1.3)

```bash
cat > hello.c <<'EOF'
#include <stdio.h>
int answer = 42;
int main(void){ printf("answer=%d\n", answer); return 0; }
EOF
gcc -E hello.c -o hello.i        # ① preprocess
gcc -S hello.i -o hello.s        # ② compile
gcc -c hello.s -o hello.o        # ③ assemble
gcc hello.o -o hello             # ④ link
./hello
```

**Observe:** `wc -l hello.i` (header explosion); `hello.s` is human-readable
assembly; `file hello.o` says "relocatable"; `file hello` says "pie executable".

**Checkpoint questions:**
1. Why is `hello.i` thousands of lines?
2. What ELF type is `hello.o` vs `hello`, and why does `file` call `hello` a
   "shared object"?

---

## Lab 1 — Decode an ELF header by hand

**Goal:** read raw header bytes and verify against `readelf`. (Part 4.2)

```bash
xxd -g1 -l 64 hello
readelf -h hello
```

**Tasks:**
1. Find `EI_CLASS`, `EI_DATA`, `e_type`, `e_machine`, `e_entry` in the hex.
2. Confirm `e_entry` matches `readelf -h`. Remember little-endian (reverse the
   bytes).
3. Disassemble the entry point: `objdump -d hello | grep -A10 '<_start>'`. Is it
   `main`? (No — it's `_start`. Part 5.5.)

---

## Lab 2 — Sections vs segments (the dual nature)

**Goal:** internalize the linking view vs execution view. (Parts 4.1, 4.3, 4.4)

```bash
readelf -S hello          # sections (linking view)
readelf -l hello          # segments (execution view) + section-to-segment map
```

**Tasks:**
1. Which sections are in the executable (R-E) `PT_LOAD`? Which in the writable
   one?
2. Find a `PT_LOAD` where `MemSiz > FileSiz`. Which section causes it? (`.bss`.)
3. Which sections are in **no** segment? Why don't they need to be loaded?
4. Strip and confirm it still runs: `cp hello h2 && strip --strip-all h2 &&
   ./h2 && readelf -l h2` — segments intact, far fewer sections.

---

## Lab 3 — Symbols, binding, and the two tables

**Goal:** understand defined/undefined, local/global/weak, symtab vs dynsym.
(Parts 3.2, 4.5)

```bash
nm hello | grep -E 'main|answer|printf'
nm -D hello | head
readelf -sW hello | grep -E 'main|answer|printf'
```

**Tasks:**
1. What is `printf`'s `Ndx`? (`UND` — imported.) Its binding?
2. What section is `answer` in? `main`?
3. Make `answer` static (`static int answer = 42;`), recompile, and check its
   binding changes GLOBAL → LOCAL.
4. Strip and show `nm hello` reports "no symbols" but `nm -D hello` still works.

---

## Lab 4 — Relocations from `.o` to executable

**Goal:** watch a relocation get created and applied. (Parts 3.3, 4.6)

```bash
cat > rel.c <<'EOF'
extern int ext;
int read_ext(void){ return ext; }
EOF
gcc -c rel.c -o rel.o
objdump -dr rel.o            # see the placeholder + R_X86_64_PC32
readelf -rW rel.o
```

**Tasks:**
1. Identify the relocation type, symbol, and addend. Why is the addend `-4`?
2. Define `ext` and link: `printf 'int ext=7;\n' > ext.c && gcc rel.o ext.c -o
   rel && objdump -d rel | grep -A2 read_ext`. The placeholder is now a real
   RIP-relative displacement. Compute it by hand and verify (§4.6.3).

---

## Lab 5 — The link-order footgun

**Goal:** feel why archive scan order matters. (Part 5.2)

```bash
printf '#include <math.h>\nint main(){return (int)sqrt(2.0);}\n' > m.c
gcc -c m.c
gcc -lm m.o -o bad   2>&1 || echo "FAILED: wrong order"
gcc m.o -lm -o good  && echo "OK: library after its user"
```

**Tasks:**
1. Explain the error using the left-to-right on-demand archive scan.
2. Build a tiny static lib and confirm only used members are pulled in:
   ```bash
   printf 'int used(){return 1;}\n'   > u.c
   printf 'int unused(){return 2;}\n' > n.c
   gcc -c u.c n.c && ar rcs libuu.a u.o n.o
   printf 'int used();int main(){return used();}\n' > mu.c
   gcc mu.c -L. -luu -o mu -Wl,-Map=mu.map
   nm mu | grep -E 'used|unused'      # unused() should be ABSENT
   ```

---

## Lab 6 — PLT/GOT and lazy binding live

**Goal:** watch a GOT slot get patched on first call. (Part 5.4)

```bash
cat > plt.c <<'EOF'
#include <stdio.h>
int main(){ puts("first"); puts("second"); return 0; }
EOF
gcc -no-pie -O0 plt.c -o plt
objdump -d -j .plt plt | head -20
objdump -R plt | grep puts        # R_X86_64_JUMP_SLOT
```

**Tasks (in gdb):**
```
gdb -q plt
break main
run
disassemble puts@plt        # note the GOT slot address it jumps through
x/gx <got_slot_addr>        # BEFORE first puts: points back into the PLT stub
# step over the first puts call, then:
x/gx <got_slot_addr>        # AFTER: now the real libc puts address
```
1. Confirm the slot changes after the first call (lazy binding).
2. Re-run with `LD_BIND_NOW=1 ./plt` and explain why the slot is resolved before
   `main` (eager binding, §5.4.7).

---

## Lab 7 — Build and version a shared library

**Goal:** soname, DT_NEEDED, RUNPATH, the search algorithm. (Part 5.3)

```bash
printf 'int square(int x){return x*x;}\n' > mylib.c
gcc -fPIC -c mylib.c
gcc -shared -Wl,-soname,libmylib.so.1 mylib.o -o libmylib.so.1.0.0
ln -sf libmylib.so.1.0.0 libmylib.so.1
ln -sf libmylib.so.1 libmylib.so
printf '#include <stdio.h>\nint square(int);int main(){printf("%%d\\n",square(7));}\n' > use.c
gcc use.c -L. -lmylib -Wl,-rpath,'$ORIGIN' -o use
readelf -d use | grep -E 'NEEDED|RUNPATH'
./use
```

**Tasks:**
1. Confirm `DT_NEEDED` is `libmylib.so.1` (the soname), not `libmylib.so`.
2. Use `LD_DEBUG=libs ./use 2>&1 | grep mylib` to watch the search.
3. Delete the `libmylib.so.1` symlink and run — observe the "cannot open shared
   object" error, then restore it.

---

## Lab 8 — DWARF line table & DIEs

**Goal:** map addresses to lines and read the type tree. (Parts 6.2, 6.3)

```bash
gcc -g -O0 hello.c -o hellog
readelf --debug-dump=decodedline hellog | head
addr2line -e hellog -f 0x$(nm hellog | awk '/ T main/{print $1}' | tail -c 5)
llvm-dwarfdump --debug-info hellog | sed -n '1,40p'
```

**Tasks:**
1. Find which source line `main`'s first instruction maps to.
2. In the DIE tree, find the DIE for `answer` and its `DW_AT_type` and
   `DW_AT_location`.
3. Add a `struct { int a; double b; }` global, recompile with `-g`, and read its
   member offsets from the DIEs. Confirm the padding (§1.2.5).

---

## Lab 9 — CFI, unwinding & C++ exceptions

**Goal:** see `.eh_frame` enable backtraces and exception cleanup. (Part 6.4)

```bash
cat > exc.cpp <<'EOF'
#include <cstdio>
struct Guard{ ~Guard(){ puts("~Guard"); } };
void c(){ Guard g; throw 42; }
void b(){ c(); }
void a(){ b(); }
int main(){ try{ a(); } catch(int e){ printf("caught %d\n", e); } }
EOF
g++ -O2 -g exc.cpp -o exc      # -O2 omits frame pointer → must use CFI
./exc
readelf --debug-dump=frames exc | head -30
gdb -batch -ex 'break c' -ex run -ex bt exc 2>/dev/null
```

**Tasks:**
1. Confirm the backtrace shows `c → b → a → main` even at `-O2` (no frame
   pointer) — proving `.eh_frame` CFI works.
2. Confirm `~Guard` runs during the throw (phase-2 cleanup via the LSDA).
3. Build with `-fno-exceptions` (remove the throw) and compare `.eh_frame` size.

---

## Lab 10 — C++ mangling, vtables & the "undefined vtable" error

**Goal:** decode mangling and reproduce the key-function bug. (Part 7.3)

```bash
cat > poly.cpp <<'EOF'
struct Base { virtual void f(); virtual ~Base(){} };
void Base::f(){}                  // key function defined → vtable emitted
struct Derived : Base { void f() override {} };
int main(){ Base* p = new Derived(); p->f(); delete p; return 0; }
EOF
g++ poly.cpp -o poly && ./poly && echo OK
nm -C poly | grep -E 'vtable|typeinfo|Base::f'
echo '_ZTV4Base' | c++filt          # "vtable for Base"
```

**Tasks:**
1. Decode `_ZN4Base1fEv` and `_ZTV7Derived` by hand, then check with `c++filt`.
2. **Reproduce the bug:** delete the line `void Base::f(){}` (declare `f` but
   never define it). Recompile → `undefined reference to vtable for Base`.
   Explain via the key-function rule (§7.3.3). Restore the definition.
3. Inspect COMDAT groups: `g++ -c poly.cpp && readelf -gW poly.o`.

---

## Lab 11 — Static vs dynamic vs LTO sizes

**Goal:** measure the trade-offs you've read about. (Parts 5.1, 5.3, 7.2)

```bash
gcc hello.c -o dyn
gcc -static hello.c -o sta
gcc -O2 -flto hello.c -o lto
ls -l dyn sta lto
ldd dyn ; ldd sta 2>&1 | head -1     # sta: "not a dynamic executable"
readelf -l sta | grep INTERP || echo "static: no PT_INTERP"
size dyn sta
```

**Tasks:**
1. Explain the size difference (static bundles libc).
2. Confirm the static binary has no `PT_INTERP` and no `DT_NEEDED`.
3. For a multi-file program, verify LTO inlines across TUs (use the §7.2.5
   example) by diffing `objdump -d` of LTO vs non-LTO.

---

## Lab 12 — Security posture audit

**Goal:** read and change a binary's hardening. (Part 7.4)

```bash
gcc hello.c -o h_def                                  # distro defaults
gcc -no-pie -fno-stack-protector -z norelro -z lazy hello.c -o h_weak
for f in h_def h_weak /bin/ls; do
  echo "== $f =="
  readelf -hld "$f" 2>/dev/null | grep -E 'Type:|GNU_STACK|RELRO|BIND_NOW|FLAGS'
done
# if available:  checksec --file=h_def ; checksec --file=h_weak
```

**Tasks:**
1. Compare PIE, RELRO, BIND_NOW, NX between `h_def` and `h_weak`.
2. Count `endbr64` landing pads with/without `-fcf-protection=full`:
   `gcc -fcf-protection=full hello.c -o h_cet && objdump -d h_cet | grep -c
   endbr64`.

---

## Capstone — Write a minimal ELF parser

**Goal:** prove mastery by reading ELF without `readelf`.

Write a program (C, Python, or Rust) that, given an ELF64 file:
1. Validates the magic and prints class/endianness/type/machine/entry.
2. Lists sections (name via `.shstrtab`, type, flags, addr, size).
3. Lists program headers (type, flags, vaddr, filesz, memsz).
4. Lists `.symtab`/`.dynsym` symbols (name via the linked strtab, bind, type,
   shndx).
5. **Bonus:** decode `.rela.text` and print `type S+A-P`-style descriptions.

Reference structs: §4.2 (Ehdr), §4.3 (Shdr), §4.4 (Phdr), §4.5 (Sym), §4.6
(Rela). Validate your output against `readelf -h/-S/-l/-s/-r`. A starter in
Python:

```python
import struct, sys
data = open(sys.argv[1], 'rb').read()
assert data[:4] == b'\x7fELF', "not ELF"
is64 = data[4] == 2; le = '<' if data[5] == 1 else '>'
# Elf64_Ehdr: 16s H H I Q Q Q I H H H H H H
e = struct.unpack_from(le+'16sHHIQQQIHHHHHH', data, 0)
print("type=%d machine=%#x entry=%#x shoff=%d shnum=%d shstrndx=%d"
      % (e[1], e[2], e[4], e[6], e[12], e[13]))
# ... walk section headers at e[6], stride 64, count e[12]; resolve names via
#     the .shstrtab at index e[13]; then symbols and relocations.
```

When this parser agrees with `readelf` across several binaries, you understand
ELF.

---

## Where to go next

- **Specs:** the System V gABI, the x86-64 psABI, the DWARF 5 standard, the
  Itanium C++ ABI. Now that you have the mental model, they're readable.
- **Source:** glibc's `elf/rtld.c` and `elf/dl-*.c` (the dynamic loader),
  binutils `bfd`/`ld`, LLVM `lld` and the MC layer, `libdwarf`/`libunwind`.
- **Books:** *Linkers and Loaders* (John Levine); *Computer Systems: A
  Programmer's Perspective* (Bryant & O'Hallaron), chapter 7.
- See [99-reference/cheatsheets.md](../99-reference/cheatsheets.md) and
  [99-reference/glossary.md](../99-reference/glossary.md).
