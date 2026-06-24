# 7.5 — The Binary-Analysis Toolbox

You've learned the formats and mechanisms; this chapter is the practitioner's
reference for the tools that observe them, organized by task, with the exact
flags experts reach for. Bookmark it.

---

## 7.5.1 The toolbox at a glance

```
   QUESTION                                  TOOL
   ────────                                  ────
   "What kind of file is this?"              file
   "Show me the ELF structure"               readelf -a
   "Disassemble the code"                    objdump -d / -dr  /  llvm-objdump
   "What symbols does it define/need?"       nm / nm -D / -C
   "Demangle this C++ name"                  c++filt
   "Which libraries does it need?"           ldd / readelf -d
   "How big are the sections?"               size
   "What strings are in it?"                 strings
   "Map an address to source"                addr2line
   "Show DWARF debug info"                   llvm-dwarfdump / readelf --debug-dump
   "Trace syscalls / lib calls"              strace / ltrace
   "Interactive debugging"                   gdb / lldb
   "Edit/transform a binary"                 objcopy / strip / patchelf
   "Audit security hardening"                checksec
   "Deep RE / scriptable analysis"           radare2 / rizin / Ghidra
```

---

## 7.5.2 `readelf` — the ELF spec made visible

The single most important tool. It speaks the ELF spec field names literally.

```bash
readelf -h FILE     # ELF header (§4.2)
readelf -S FILE     # section headers (§4.3)        -W = wide, don't truncate
readelf -l FILE     # program headers + seg mapping (§4.4)
readelf -s FILE     # .symtab and .dynsym (§4.5)
readelf -d FILE     # .dynamic array (§5.3)
readelf -r FILE     # relocations (§4.6)
readelf -n FILE     # notes (build-id, ABI tag, GNU properties)
readelf -V FILE     # symbol versioning (§4.5.7)
readelf -g FILE     # section groups / COMDAT (§7.3.4)
readelf -x .rodata FILE   # hex dump a section
readelf -p .interp FILE   # string dump a section
readelf --debug-dump=info|decodedline|frames|abbrev FILE   # DWARF (Part 6)
readelf -a FILE     # everything
```

> Prefer `readelf` over `objdump -h` for *structure* — it follows the spec
> exactly and handles every field. Use `-W` always to avoid truncated columns.

---

## 7.5.3 `objdump` — disassembly and BFD views

```bash
objdump -d FILE                 # disassemble executable sections
objdump -D FILE                 # disassemble ALL sections (incl. data — careful)
objdump -dr FILE                # disassembly WITH relocations interleaved (key!)
objdump -d -M intel FILE        # Intel syntax
objdump -d --insn-width=12 FILE # show full instruction bytes
objdump -t FILE                 # symbol table (BFD view)
objdump -T FILE                 # dynamic symbol table
objdump -R FILE                 # dynamic relocations
objdump -s -j .rodata FILE      # full contents of a section
objdump --dwarf=frames FILE     # CFI
objdump -C ...                  # demangle C++ names in output
objdump -d -l FILE              # interleave source line numbers (needs -g)
objdump -S FILE                 # interleave SOURCE CODE with disassembly (needs -g)
```

> `objdump -dr` is the workhorse for understanding relocations (§2.2.5): it shows
> the placeholder bytes *and* the relocation that fills them, side by side.
> `objdump -S` (source-interleaved disassembly) is invaluable for seeing what the
> compiler did to each line.

LLVM equivalents (`llvm-objdump`, `llvm-readelf`, `llvm-nm`) are drop-in,
sometimes clearer, and required for some LLVM-specific info.

---

## 7.5.4 `nm` — symbols fast

```bash
nm FILE                # symbols from .symtab
nm -D FILE             # dynamic symbols (.dynsym) — survives strip
nm -C FILE             # demangle C++
nm -n FILE             # sort by address
nm --size-sort FILE    # sort by symbol size (find the bloat)
nm -A *.o              # prefix each line with the filename (which .o has it?)
nm -u FILE             # only undefined symbols
```

The symbol **type letter** (column 2):

```
   T/t  in .text (code)        T=global, t=local
   D/d  in .data               B/b  in .bss
   R/r  in .rodata             W/w  weak symbol
   U    undefined (needs resolution)
   A    absolute               C    common
   i/I  indirect (IFUNC)       V/v  weak object
```

```bash
nm -C app | grep ' U '     # everything this object still needs (undefined)
nm -C lib.a | grep ' T '   # everything an archive provides
```

---

## 7.5.5 Dynamic linking observation

```bash
ldd FILE                          # resolved shared-lib dependencies (DON'T run on
                                  # untrusted binaries — it may execute the loader)
readelf -d FILE | grep NEEDED     # safer: just the DT_NEEDED list
LD_DEBUG=help ./app               # list LD_DEBUG categories
LD_DEBUG=libs ./app               # trace library search & loading (§5.3.5)
LD_DEBUG=bindings ./app           # trace symbol binding (which def wins)
LD_DEBUG=reloc ./app              # trace relocation processing
LD_DEBUG=statistics ./app         # relocation counts & startup timing
LD_SHOW_AUXV=1 ./app              # dump the auxiliary vector (§5.5.2)
LD_BIND_NOW=1 ./app               # force eager binding (test BIND_NOW behavior)
LD_PRELOAD=./hook.so ./app        # interpose functions (§5.2.6)
ltrace ./app                      # trace library calls (watch PLT resolution)
```

---

## 7.5.6 GDB recipes for toolchain internals

```bash
gdb ./app
```

```
   # Where am I in the binary?
   info inferiors / info proc mappings    # the loaded segments (vmmap)
   info sharedlibrary                     # loaded .so's and their load addresses
   info files                             # sections and their runtime addresses

   # Symbols & addresses
   info address main                      # symbol → address
   info line *0x1149                      # address → source line (DWARF line prog)
   info functions regex                   # find functions
   info scope main                        # variables in scope + their locations (CFI/loc)

   # The GOT/PLT in action (§5.4)
   disassemble puts@plt
   x/3i 'puts@plt'
   p (void*) *(void**)&got_slot           # inspect a GOT slot before/after first call

   # Unwinding (§6.4)
   bt                                     # backtrace (uses .eh_frame)
   bt full                                # + locals per frame
   info frame                             # CFA, saved regs, return addr for this frame

   # Watch dynamic loading
   set stop-on-solib-events 1
   catch load                             # break when a library is loaded
   break _dl_runtime_resolve              # break in the lazy resolver
```

`lldb` (default on macOS) has equivalent commands (`image list`, `disassemble`,
`bt`, `frame variable`).

---

## 7.5.7 Transforming binaries: objcopy, strip, patchelf

```bash
# Stripping (§4.4.8)
strip --strip-all FILE              # remove .symtab + debug + non-essential
strip --strip-debug FILE            # remove only debug, keep symbols
strip --strip-unneeded lib.so       # for shared libs (keep dynsym)

# Split debug info (§6.1.6)
objcopy --only-keep-debug FILE FILE.debug
strip --strip-debug FILE
objcopy --add-gnu-debuglink=FILE.debug FILE

# Extract / inject sections
objcopy -O binary -j .text FILE text.bin     # dump raw .text
objcopy --add-section .foo=data.bin FILE FILE2
objcopy --set-section-flags ...

# Format conversion (embedded)
objcopy -O ihex firmware.elf firmware.hex
objcopy -O binary firmware.elf firmware.bin

# Patch a binary's dynamic info WITHOUT relinking (patchelf)
patchelf --set-interpreter /path/ld.so FILE     # change PT_INTERP
patchelf --set-rpath '$ORIGIN/lib' FILE         # change RUNPATH (§5.3.5)
patchelf --replace-needed libold.so libnew.so FILE
patchelf --print-needed FILE
```

> `patchelf` is the surgical tool for relocating/repackaging prebuilt binaries
> (common in build systems like Nix and when bundling third-party `.so`s).

---

## 7.5.8 Deep reverse-engineering platforms

```
   radare2 / rizin : scriptable CLI RE framework.
                     r2 -A FILE ; aaa ; afl (list funcs) ; pdf @ main (disasm) ;
                     iS (sections) ; ii (imports) ; ir (relocs) ; V (visual)
   Ghidra          : NSA's decompiler — turns machine code back into C-like
                     pseudocode; best for understanding stripped binaries.
   Binary Ninja / IDA : commercial; powerful disassembly/decompilation.
   pwntools (Python): exploit dev + ELF parsing (from pwn import ELF; ELF('app'))
   capstone/keystone : disassembler/assembler libraries for your own tools.
   LIEF             : Python/C++ library to parse & MODIFY ELF/PE/Mach-O programmatically.
```

**Try it ▸** A two-line decompile-ish view with radare2:

```bash
r2 -q -c 'aaa; pdf @ sym.main' ./demo 2>/dev/null   # analyze + disassemble main
```

---

## 7.5.9 A 60-second "explain this binary" routine

When handed an unknown binary, this sequence orients you fast:

```bash
F=./mystery
file "$F"                                   # type, arch, PIE?, stripped?
readelf -h "$F"                             # entry, type, machine
readelf -d "$F" | grep -E 'NEEDED|RUNPATH|SONAME|FLAGS'   # deps & posture
readelf -l "$F" | grep -E 'INTERP|GNU_STACK|RELRO'        # interp & hardening
nm -DC "$F" 2>/dev/null | grep ' U ' | head # imported (undefined) symbols
nm -DC "$F" 2>/dev/null | grep ' T ' | head # exported functions
strings -n 8 "$F" | head -40                # human-readable clues
checksec --file="$F" 2>/dev/null            # mitigations summary
objdump -d "$F" | sed -n '/<main>:/,/ret/p' # what does main do?
```

This single routine exercises nearly every concept in this guide.

---

## Summary

- `readelf` is the canonical structure dumper (use `-W`); `objdump -dr`/`-S` is
  the disassembly + relocation + source workhorse; `nm`/`nm -D`/`-C` answer
  symbol questions.
- Dynamic behavior is observable live via `LD_DEBUG=*`, `LD_SHOW_AUXV`,
  `LD_PRELOAD`, `ltrace`, and `strace`.
- GDB/LLDB expose loaded segments, GOT/PLT slots, line mappings, and CFI-driven
  backtraces interactively.
- `objcopy`/`strip`/`patchelf` transform binaries (split debug, change interp/
  rpath, convert formats) without recompiling.
- `radare2`/`Ghidra`/`LIEF`/`pwntools` cover deep RE and programmatic
  manipulation.
- The 60-second routine in §7.5.9 is a practical capstone exercising the whole
  guide.

This completes Part 7. → Part 8 puts it all into practice:
[Guided labs & exercises](../08-labs/exercises.md)
