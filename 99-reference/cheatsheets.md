# Cheat Sheets

Dense, printable quick-reference for everything in the guide.

---

## ELF structures (64-bit) — sizes & key fields

```
 Elf64_Ehdr  (64 B)   magic, EI_CLASS/DATA, e_type, e_machine, e_entry,
                      e_phoff, e_shoff, e_phnum, e_shnum, e_shstrndx
 Elf64_Shdr  (64 B)   sh_name, sh_type, sh_flags, sh_addr, sh_offset, sh_size,
                      sh_link, sh_info, sh_addralign, sh_entsize
 Elf64_Phdr  (56 B)   p_type, p_flags, p_offset, p_vaddr, p_paddr,
                      p_filesz, p_memsz, p_align
 Elf64_Sym   (24 B)   st_name, st_info(bind|type), st_other(vis), st_shndx,
                      st_value, st_size
 Elf64_Rela  (24 B)   r_offset, r_info(sym<<32|type), r_addend
 Elf64_Dyn   (16 B)   d_tag, d_un(d_val|d_ptr)
```

## ELF file types
```
 ET_REL(1)=.o   ET_EXEC(2)=fixed exe   ET_DYN(3)=.so or PIE   ET_CORE(4)=dump
```

## Section flags / Segment perms
```
 sh_flags:  A=ALLOC  W=WRITE  X=EXEC  M=MERGE  S=STRINGS  T=TLS  G=GROUP
 p_flags:   R=read   W=write  E=exec
```

## Symbol decode
```
 bind = st_info>>4 :  LOCAL(0) GLOBAL(1) WEAK(2) GNU_UNIQUE(10)
 type = st_info&0xf:  NOTYPE(0) OBJECT(1) FUNC(2) SECTION(3) FILE(4) TLS(6) IFUNC(10)
 vis  = st_other&3 :  DEFAULT(0) INTERNAL(1) HIDDEN(2) PROTECTED(3)
 shndx special    :  UNDEF(0) ABS(0xfff1) COMMON(0xfff2)
```

## nm type letters
```
 T/t .text  D/d .data  B/b .bss  R/r .rodata  U undef  W/w weak  A abs  C common
 (uppercase = global, lowercase = local)
```

---

## x86-64 System V ABI (the must-knows)

```
 INT args (in order):  RDI RSI RDX RCX R8 R9   then stack (right-to-left)
 FP  args:             XMM0..XMM7   (AL = #vector regs for varargs)
 Return:               RAX (+RDX) ; XMM0 (+XMM1)
 Callee-saved:         RBX RBP R12 R13 R14 R15 RSP
 Caller-saved:         RAX RCX RDX RSI RDI R8 R9 R10 R11, all XMM
 Stack:                grows down; 16-byte aligned at `call`; 128-B red zone
 Aggregates:           <=16B in regs (per-eightbyte class); >16B / non-trivial
                       C++ → memory ("sret" hidden pointer in RDI, result in RAX)
```

## AArch64 (AAPCS64)
```
 INT args: X0..X7  FP: V0..V7   return X0 / V0   indirect-result: X8
 Callee-saved: X19..X28, V8..V15(low64)   LR=X30  FP=X29   thread ptr: TPIDR_EL0
```

---

## Addressing & encoding (x86-64)

```
 mem operand:  base + index*scale + disp        (scale ∈ {1,2,4,8})
 AT&T:         disp(base, index, scale)          Intel: [base+index*scale+disp]
 RIP-relative: sym(%rip)  /  [rip + sym]         (PIC; reloc lives in the disp32)
 instr layout: [prefixes][REX][opcode][ModR/M][SIB][disp][imm]   (<=15 B)
 REX 0x4WRXB:  W=64-bit  R/X/B extend reg/index/base to r8..r15
 lea = compute address (NO memory access)   xor reg,reg = zero idiom
```

---

## Relocation formulas (x86-64)

```
 R_X86_64_64        S + A          64-bit absolute
 R_X86_64_32/32S    S + A          32-bit absolute (zero/sign ext)
 R_X86_64_PC32      S + A - P      PC-relative (calls, RIP data); A usually -4
 R_X86_64_PLT32     L + A - P      call via PLT
 R_X86_64_GOTPCREL  G + GOT + A-P  RIP-relative load of GOT entry
 --- dynamic (applied by ld.so) ---
 R_X86_64_RELATIVE  B + A          add load base (PIE internal pointers)
 R_X86_64_GLOB_DAT  S              fill a GOT data slot
 R_X86_64_JUMP_SLOT S              fill a PLT GOT slot (lazy/now)
 R_X86_64_COPY      copy Z bytes   copy reloc (legacy data import)
 R_X86_64_IRELATIVE resolver()     IFUNC dispatch
 (S=sym addr, A=addend, P=patch addr, B=base, L=PLT entry, G=GOT offset)
```

---

## GAS directives (linker-relevant)

```
 .text .data .bss .section NAME,"FLAGS",@TYPE   .pushsection/.popsection
 .globl x  .weak x  .local x  .hidden x   .type x,@function|@object  .size x,.-x
 .byte .word .long/.int .quad   .ascii .asciz/.string   .zero .space
 .p2align N (=2^N)   .balign N (=N bytes)    .comm x,size,align  .lcomm
 .cfi_startproc/.cfi_def_cfa/.cfi_offset/.cfi_endproc   (→ .eh_frame)
 @PLT @GOTPCREL @GOTOFF @TPOFF @TLSGD @GOTTPOFF   (relocation selectors)
```

---

## .dynamic tags (DT_*)
```
 NEEDED(lib) SONAME RPATH/RUNPATH  STRTAB SYMTAB GNU_HASH
 RELA RELASZ RELAENT  JMPREL PLTRELSZ PLTREL PLTGOT  RELR RELRSZ
 INIT FINI INIT_ARRAY[SZ] FINI_ARRAY[SZ]  FLAGS/FLAGS_1(BIND_NOW,PIE,..)  NULL
```

## Program-header types (PT_*)
```
 LOAD INTERP DYNAMIC PHDR NOTE TLS
 GNU_EH_FRAME(→.eh_frame_hdr) GNU_STACK(perms) GNU_RELRO GNU_PROPERTY
```

---

## DWARF sections & concepts
```
 .debug_info   DIE tree        .debug_abbrev   DIE schemas
 .debug_str    name pool       .debug_line     addr↔line bytecode program
 .debug_loclists var locations .debug_rnglists  scope ranges
 .debug_addr   addr pool(v5)   .debug_aranges   addr→CU index
 .debug_names/.gdb_index name→DIE   .eh_frame/.debug_frame  CFI (unwinding)
 DIE = TAG (DW_TAG_*) + ATTRS (DW_AT_*) + children
 location = DWARF expr (DW_OP_*); optimized vars → per-PC location lists
 CFI: CIE(template)+FDE(per func); CFA = caller's RSP@call; .cfi_* → .eh_frame
 C++ throw: .eh_frame (how to unwind) + LSDA/.gcc_except_table (what) + personality
```

---

## C++ mangling quick decode (Itanium)
```
 _Z          prefix        N..E  nested(qualified)     <n><id>  length-prefixed name
 i j l c d f b v = int unsigned long char double float bool void
 P=ptr R=lref O=rref K=const V=volatile   S_/S0_.. = substitution backref
 I..E = template args   T_ = first template param
 _ZTV=vtable _ZTI=typeinfo _ZTS=typeinfo-name _ZThn=thunk _ZGV=static guard
 __cxa_throw/__cxa_atexit/__cxa_guard_*/__cxa_pure_virtual   _GLOBAL__sub_I_*=static init
 decode any with:  echo '<sym>' | c++filt
```

---

## Command quick-reference
```
 file F                       type/arch/PIE/stripped
 readelf -hSlsdrnVg -W F      header/sections/segs/syms/dyn/relocs/notes/ver/groups
 readelf --debug-dump=info|decodedline|frames|abbrev F   DWARF
 objdump -dr / -S / -M intel  disasm+relocs / +source / Intel syntax
 nm / nm -D / -C / -n / --size-sort   symbols
 c++filt SYM                  demangle
 ldd F  /  readelf -d F       shared deps
 addr2line -e F -f -i ADDR    addr → func+file:line (inlined too)
 size F   strings F           section sizes / printable strings
 LD_DEBUG=libs|bindings|reloc|statistics ./app    trace the loader
 LD_SHOW_AUXV=1 ./app         auxv    LD_BIND_NOW=1   eager bind   LD_PRELOAD=x.so
 strace -e execve,mmap,openat ./app   ltrace ./app
 objcopy / strip / patchelf   transform binaries
 checksec --file=F            hardening audit
 r2 -A F   (aaa; afl; pdf @ main; iS; ii; ir)     reverse engineering
```

## Build flags quick-reference
```
 -E/-S/-c            stop after preprocess/compile/assemble
 -g  -gdwarf-5  -gsplit-dwarf   debug info / version / fission
 -O0/-Og/-O2/-O3/-Os            optimization (-Og = debug-friendly)
 -fPIC -fPIE -pie / -no-pie     position independence
 -ffunction-sections -fdata-sections + -Wl,--gc-sections   dead-code strip
 -flto / -flto=thin             link-time optimization
 -fvisibility=hidden            hide symbols (shared-lib exports)
 -static                        bundle libc (no ld.so)
 -fstack-protector-strong  -D_FORTIFY_SOURCE=3  -fcf-protection=full   hardening
 -Wl,-z,relro -Wl,-z,now        full RELRO + eager binding
 -Wl,-rpath,'$ORIGIN/../lib'    relocatable runtime search path
 -Wl,-Map=out.map               linker map (where everything landed)
 -fuse-ld=lld|mold              pick a faster linker
 -masm=intel                    Intel-syntax assembly output
```
