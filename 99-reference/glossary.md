# Glossary

Concise definitions of every key term in the guide, with the section that covers
it in depth.

---

**ABI (Application Binary Interface)** — The binary-level contract (calling
convention, type layout, name mangling, exception model) that lets separately
compiled code interoperate. (§2.4)

**Addend** — The constant `A` in a relocation formula; in RELA it lives in the
record, in REL in the patched field. (§3.3, §4.6)

**Address point** — The location within a vtable that the object's vptr points
at (the first virtual function pointer; RTTI/offset-to-top are at negative
offsets). (§7.3.3)

**Alignment** — Requirement that an object reside at an address divisible by its
alignment; drives padding in structs and between sections. (§1.2.5)

**Archive (`.a`)** — A bundle of `.o` files (made with `ar`); the linker pulls in
only members that resolve pending undefined symbols. (§5.2.3)

**ASLR** — Address Space Layout Randomization; randomizes load addresses each run
(requires PIE for the executable itself). (§7.4.3)

**Auxiliary vector (auxv)** — Kernel→userspace key/value array on the initial
stack (`AT_PHDR`, `AT_BASE`, `AT_ENTRY`, `AT_RANDOM`, `AT_HWCAP`). (§5.5.2)

**.bss** — Zero-initialized data section; `SHT_NOBITS` (size only, no file
bytes). (§3.2, §4.3)

**BIND_NOW** — Resolve all dynamic symbols at load time (eager binding) instead
of lazily; enables Full RELRO. (§5.4.7)

**Build-id** — A hash in `.note.gnu.build-id` linking a stripped binary to its
separate debug file (used by debuginfod). (§6.1.6)

**Caller-/Callee-saved** — Register preservation responsibility split: callee
must restore callee-saved (RBX,RBP,R12–R15); caller must save caller-saved if
needed across a call. (§2.4.4)

**Canary (stack cookie)** — Random value between locals and the return address;
checked before return to detect overflow. Lives in TLS (`%fs:0x28`). (§7.4.4)

**CFA (Canonical Frame Address)** — The anchor for unwinding: the caller's RSP at
the call site; all saved registers are expressed relative to it. (§6.4.2)

**CFI (Call Frame Information)** — DWARF data describing how to unwind each frame;
encoded as CIE+FDE in `.eh_frame`/`.debug_frame`. (§6.4)

**CIE / FDE** — Common Information Entry (shared unwind template) / Frame
Description Entry (per-function unwind program). (§6.4.4)

**Code model** — Compiler assumption about code/data address range
(`small`/`medium`/`large`) determining whether 32-bit relocations suffice.
(§3.3.8)

**COMDAT group** — A section group (`SHT_GROUP`) keyed by a signature; the linker
keeps one copy across TUs (deduplicates inline/template definitions). (§7.3.4)

**Common symbol** — A tentative C definition (`SHN_COMMON`); multiple are merged.
`-fno-common` (modern default) disables this. (§3.2.6)

**CET** — Intel Control-flow Enforcement: IBT (`endbr64` landing pads) + Shadow
Stack; ARM analogues are BTI/PAC. (§7.4.6)

**.data / .rodata** — Initialized writable / read-only data sections. (§3.2)

**.debug_* sections** — DWARF debug info; not loaded at runtime (except
`.eh_frame`). (§6.1)

**DIE (Debugging Information Entry)** — A node in `.debug_info` describing a
function/variable/type/scope; has a tag + attributes + children. (§6.2)

**DTV (Dynamic Thread Vector)** — Per-thread array mapping module id → that
thread's TLS block, used by `__tls_get_addr`. (§7.1.5)

**.dynamic** — The dynamic linking control array (`Elf64_Dyn` tag/value pairs)
that drives `ld.so`. (§5.3.3)

**.dynsym / .dynstr** — Runtime (loaded) dynamic symbol table and its strings;
survive stripping. Contrast `.symtab`/`.strtab`. (§4.5.4)

**Eightbyte** — An 8-byte chunk used to classify aggregates for ABI argument
passing (INTEGER/SSE/MEMORY). (§2.4.6)

**.eh_frame / .eh_frame_hdr** — Loaded CFI tables for runtime unwinding (C++
exceptions) + a binary-search index over them. (§6.4)

**ELF (Executable and Linkable Format)** — The container format for objects,
executables, shared libraries, and core dumps. (Part 4)

**Endianness** — Byte order of multi-byte values; x86-64/ARM are little-endian.
(§1.2.2)

**Entry point (`e_entry`)** — Address where execution begins (`_start`, not
`main`). (§4.2.3, §5.5.4)

**Forward reference** — A symbol used before defined; reason assemblers need two
passes. (§3.1.3)

**Frame pointer (RBP)** — Optional stable base for a stack frame; omitted at
`-O2` (then unwinding needs CFI). (§2.4.5)

**GOT (Global Offset Table)** — Writable table of pointer slots the loader fills
so PIC can reach symbols at runtime addresses. (§5.4.2)

**.init_array / .fini_array** — Arrays of constructor / destructor pointers run
before `main` / at exit (C++ global ctors/dtors). (§5.5.5)

**Interposition** — First-definition-wins symbol override in dynamic linking;
basis of `LD_PRELOAD`. (§5.2.6)

**IFUNC** — Indirect function: a resolver chosen at load time (e.g. CPU-specific
`memcpy`); uses `R_X86_64_IRELATIVE`. (§4.6.4)

**LEB128** — Little-Endian Base 128 variable-length integer encoding, pervasive
in DWARF. (§1.2.4)

**Lazy binding** — Resolve a PLT function's address on first call via
`_dl_runtime_resolve`, patching its GOT slot. (§5.4.5)

**Linker script** — Rules mapping input sections to output sections and assigning
addresses; default built-in or custom (`-T`). (§5.1.4)

**Location counter (`.`)** — The assembler's current offset within a section;
labels capture it. (§3.1.2)

**Location expression / list** — DWARF stack-machine program computing where a
variable lives (per-PC for optimized code). (§6.2.7)

**LSDA** — Language-Specific Data Area (`.gcc_except_table`): per-function
try/catch ranges and landing pads used during exception unwinding. (§6.4.6)

**LTO (Link-Time Optimization)** — Defer optimization to link time using shipped
IR for whole-program analysis; ThinLTO is the scalable variant. (§7.2)

**Mangling** — Encoding a C++ entity's full signature into a unique ELF symbol
(Itanium `_Z...` scheme). (§7.3.1)

**ModR/M / SIB** — x86 instruction bytes encoding operands and the
`base+index*scale+disp` addressing. (§2.2.2)

**NOBITS / PROGBITS** — Section occupies no file bytes (`.bss`) / has file bytes
(`.text`/`.data`). (§3.2.1)

**ODR (One Definition Rule)** — C++ rule of exactly one (consistent) definition;
mechanically shadowed by strong symbols + WEAK/COMDAT dedup; inconsistent
duplicates are silent UB. (§5.2.7, §7.3.4)

**PIC / PIE** — Position-Independent Code / Executable; uses RIP-relative and GOT
addressing so the module can load anywhere (enables ASLR). (§2.1.5, §7.4.3)

**PLT (Procedure Linkage Table)** — Read-only trampoline stubs that jump through
GOT slots to reach external functions. (§5.4)

**Program header / segment** — Execution-view descriptor (`Elf64_Phdr`); a
page-aligned, permission-homogeneous chunk the loader maps. (§4.4)

**Red zone** — 128 bytes below RSP usable by leaf functions without adjusting RSP
(System V only). (§1.1.6)

**Relaxation** — Linker/assembler shrinking instructions (e.g. branch sizing,
TLS GD→LE) once distances are known. (§3.1.5, §7.1.4)

**RELA / REL** — Relocation record with explicit addend (x86-64) / implicit
in-field addend (i386). (§4.6.1)

**Relocation** — Instruction to patch a field with `f(S,A,P,…)`; the glue of
separate compilation. (§3.3, §4.6)

**RELRO** — RELocation Read-Only: re-protect the GOT/`.dynamic` read-only after
startup; Full RELRO adds BIND_NOW. (§5.4.7, §7.4.5)

**RIP-relative addressing** — `[rip + disp32]`; addresses relative to the next
instruction, the basis of PIC. (§2.1.5)

**RUNPATH / RPATH** — Library search paths baked into a binary (RUNPATH is
checked after `LD_LIBRARY_PATH`; RPATH before). (§5.3.5)

**Section / section header** — Linking-view unit (`Elf64_Shdr`): named bytes +
flags the linker manipulates. (§4.3)

**Section group** — See COMDAT. (§7.3.4)

**Soname (DT_SONAME)** — A shared library's ABI-version name (e.g.
`libfoo.so.1`), recorded as `DT_NEEDED` in dependents. (§5.3.4)

**Static linking** — Resolving/combining at link time; with `-static`, also
bundling libc so there's no `ld.so`. (§5.1, §5.5.7)

**Strong / weak symbol** — A normal definition / an overridable one; strong beats
weak, and a missing weak resolves to 0. (§5.2.2)

**.symtab / .strtab** — Full (link-time/debug, strippable) symbol table and its
strings. (§4.5)

**String table** — NUL-separated strings referenced by byte offset (`.strtab`,
`.dynstr`, `.shstrtab`). (§4.5.6)

**Symbol** — Named value (address/constant/reference) with binding, type,
visibility, and a section. (§3.2.4, §4.5)

**Symbol versioning** — Multiple versioned definitions of one name in a library
(`name@@VER`); behind `GLIBC_x.y not found`. (§4.5.7)

**Tentative definition** — A C uninitialized file-scope variable; historically a
common symbol. (§3.2.6)

**Thread pointer** — Per-thread base register for TLS (`FS` on x86-64 Linux, `GS`
on Windows/macOS, `TPIDR_EL0` on AArch64). (§7.1.2)

**Thunk / veneer** — A small trampoline the linker inserts to bridge a call that
can't reach its target directly (far branch, `this`-adjustment). (§3.3.8, §7.3.6)

**TLS (Thread-Local Storage)** — Per-thread variable storage; template in
`.tdata`/`.tbss`, four access models. (§7.1)

**Two's complement** — Standard signed-integer representation. (§1.2.3)

**Visibility** — Symbol export control (DEFAULT/HIDDEN/PROTECTED/INTERNAL);
governs shared-library exports and optimization. (§4.5.2)

**Vptr / vtable** — Per-object pointer to / per-class array of virtual function
pointers; the virtual dispatch mechanism. (§7.3.3)

**W^X (NX)** — Memory is writable XOR executable; blocks code injection.
(§7.4.2)

**Weak symbol** — See strong/weak. (§5.2.2)
