# 2.3 — Assembly Syntax, Directives & GAS vs NASM

The bytes are the same; the *text* humans (and compilers) write to produce them
comes in dialects. Reading compiler output means reading **GNU assembler (GAS)
AT&T syntax**. Writing hand-assembly often means **NASM/Intel syntax**. And
crucially: **assembler directives** — not instructions — are how sections,
symbols, alignment, and data are declared. Directives are where the assembler
meets the linker.

---

## 2.3.1 AT&T vs Intel syntax: the cheat sheet

GCC/Clang emit **AT&T** syntax by default (`.s` files). NASM, MASM, and most
documentation use **Intel** syntax. They describe identical machine code.

| Feature              | AT&T (GAS default)        | Intel (NASM/`-masm=intel`) |
|----------------------|---------------------------|----------------------------|
| Operand order        | `op src, dst`             | `op dst, src`              |
| Register prefix      | `%rax`                    | `rax`                      |
| Immediate prefix     | `$5`, `$0x10`             | `5`, `0x10`                |
| Memory deref         | `disp(base,index,scale)`  | `[base + index*scale + disp]`|
| Operand size         | suffix: `movq`, `movl`    | from operand / `qword ptr` |
| Memory example       | `mov 8(%rbx,%rcx,4), %rax`| `mov rax, [rbx+rcx*4+8]`   |
| RIP-relative         | `sym(%rip)`               | `[rip + sym]`              |

```
 SAME INSTRUCTION, TWO SPELLINGS:

   AT&T :  movq  $5, count(%rip)
   Intel:  mov   qword ptr [rip + count], 5

   "store the 64-bit immediate 5 into the global `count`"
```

> **The reversed operand order is the #1 confusion.** In AT&T,
> `mov $1, %rax` means rax := 1. In Intel, `mov rax, 1` means the same. Always
> check which dialect a dump is in. `objdump -M intel -d` switches objdump to
> Intel; `gcc -masm=intel -S` makes the compiler emit Intel.

**Try it ▸**

```bash
printf 'int x;void f(){x=5;}\n' > t.c
gcc -O2 -S t.c -o att.s            # AT&T (default)
gcc -O2 -S -masm=intel t.c -o intel.s
diff <(grep -A3 'f:' att.s) <(grep -A3 'f:' intel.s)
```

---

## 2.3.2 The size suffixes (AT&T)

Because AT&T operands don't always reveal their width, GAS uses suffixes:

```
   b = byte   (8 bit)     movb
   w = word   (16 bit)    movw
   l = long   (32 bit)    movl
   q = quad   (64 bit)    movq
   (FP)  s = single, d = double:   movss / movsd
   sign/zero extend combine sizes: movzbl (zero-extend byte→long),
                                   movslq (sign-extend long→quad), etc.
```

Intel syntax instead uses operand keywords: `byte ptr`, `word ptr`,
`dword ptr`, `qword ptr`.

---

## 2.3.3 Anatomy of an assembly file

```asm
        .file   "hello.c"
        .text                       # ── DIRECTIVE: switch to .text section
        .globl  main                # ── DIRECTIVE: make `main` an external symbol
        .type   main, @function     # ── DIRECTIVE: mark symbol type
main:                               # ── LABEL: defines symbol `main` = here
        pushq   %rbp                # ── INSTRUCTION
        movq    %rsp, %rbp          # ── INSTRUCTION
        leaq    .LC0(%rip), %rdi    # ── INSTRUCTION (references a local label)
        call    puts@PLT            # ── INSTRUCTION (call via PLT)
        movl    $0, %eax
        popq    %rbp
        ret
        .size   main, .-main        # ── DIRECTIVE: record symbol size

        .section .rodata            # ── DIRECTIVE: switch to read-only data
.LC0:                               # ── local label (".L" prefix = local)
        .string "Hello, world!"     # ── DIRECTIVE: emit bytes + NUL
```

Three categories of lines:

1. **Labels** (`main:`, `.LC0:`) — define a symbol equal to the current
   location counter in the current section.
2. **Instructions** — encoded into machine code.
3. **Directives** (start with `.`) — commands *to the assembler*: switch
   sections, emit data, declare symbol attributes, set alignment.

> **`.L` prefix ▸** Labels beginning with `.L` are **local** — the assembler
> keeps them out of the object's symbol table (they become temporary/anonymous).
> This is how compilers generate jump targets and constant labels without
> polluting the symbol table. Strip-resistant tools still see them via
> relocations.

---

## 2.3.4 The essential directives (GAS)

These are the directives that *directly shape the object file* — the ones a
linker-focused engineer must know.

### Section control

```asm
   .text                 # code section (rx)
   .data                 # initialized writable data (rw)
   .bss                  # zero-init data (rw, occupies no file space)
   .section .rodata      # read-only data
   .section .text.hot,"ax",@progbits     # custom section with explicit flags
   .pushsection / .popsection            # save/restore current section
```

The flags string and type matter:

```
   .section NAME, "FLAGS", @TYPE
                   │         └ progbits / nobits / note / init_array / ...
                   └ a=alloc, w=write, x=execute, M=mergeable, S=strings,
                     G=group (COMDAT), T=TLS
```

### Symbol attributes

```asm
   .globl  sym           # binding: STB_GLOBAL (visible to linker across TUs)
   .weak   sym           # binding: STB_WEAK (overridable, no error if missing)
   .local  sym           # binding: STB_LOCAL
   .type   sym, @function | @object        # STT_FUNC / STT_OBJECT
   .size   sym, expr     # st_size for the symbol
   .hidden sym           # ELF visibility STV_HIDDEN
   .set    alias, target # define alias = target (symbol equate)
   .equ / .equiv         # constant/symbol definitions
```

These map *one-to-one* onto ELF symbol-table fields (Part 4.5). When you write
`.globl foo` you are setting `st_info`'s binding to `STB_GLOBAL`.

### Emitting data

```asm
   .byte   0x41               # one byte
   .word   0x1234             # 2 bytes (GAS: .word=16-bit on x86)
   .long / .int   0xdeadbeef  # 4 bytes
   .quad   0x1122334455667788 # 8 bytes
   .ascii  "abc"              # bytes, NO trailing NUL
   .asciz / .string "abc"     # bytes WITH trailing NUL
   .zero   16                 # 16 zero bytes
   .space  16, 0xAA           # 16 bytes of 0xAA
```

A `.quad somesymbol` emits 8 bytes that are the *address of* `somesymbol` — and
generates an **absolute relocation** (`R_X86_64_64`) because the address isn't
known until link time. This is exactly how vtables, jump tables, and
`.init_array` entries are built.

### Alignment

```asm
   .align  16          # align location counter (platform-dependent meaning!)
   .p2align 4          # align to 2^4 = 16 bytes (unambiguous, preferred)
   .balign 16          # align to 16 bytes (byte count, unambiguous)
```

> **Pitfall ▸** `.align` means "align to N bytes" on x86 ELF but "align to 2^N"
> on some other targets. Prefer `.p2align` / `.balign` to avoid surprises. The
> assembler fills the gap with NOPs in `.text` (multi-byte NOPs!) and zeros in
> data sections.

### Linker-relevant misc

```asm
   .comm  symbol, size, align    # common (tentative) symbol → goes to .bss-like
   .lcomm symbol, size, align    # local common
   .ident "GCC: ..."             # toolchain tag, lands in .comment
   .cfi_startproc / .cfi_def_cfa / .cfi_endproc   # generate .eh_frame CFI (Part 6.4)
```

---

## 2.3.5 Symbol references and operator suffixes (`@`)

When an instruction references a symbol, a suffix selects *which relocation* to
use — this is where assembly text controls linking behavior:

```asm
   call puts@PLT          # go through the Procedure Linkage Table (R_X86_64_PLT32)
   movq var@GOTPCREL(%rip), %rax   # load var's address from the GOT
   leaq var@TPOFF(%rax), %rax      # thread-local offset (TLS)
   .quad func@GOT                  # GOT slot
```

| Suffix          | Meaning / typical relocation                         |
|-----------------|------------------------------------------------------|
| `@PLT`          | call/jump via PLT stub → `R_X86_64_PLT32`            |
| `@GOTPCREL`     | RIP-relative load of GOT entry → `R_X86_64_GOTPCREL` |
| `@GOTOFF`       | offset from the GOT base                             |
| `@TPOFF`/`@TLSGD`/`@GOTTPOFF` | thread-local storage models (Part 7.1) |
| `@PLTOFF`       | offset of a PLT entry                                |

These suffixes are the human-readable way to request a specific linking
strategy; the assembler turns them into relocation *types*.

---

## 2.3.6 NASM in one screen (for hand-written assembly)

If you write standalone assembly, NASM (Intel syntax) is common. Key
differences from GAS:

```nasm
            section .text
            global  main
            extern  puts            ; declare external symbol

main:
            push    rbp
            mov     rbp, rsp
            lea     rdi, [rel msg]   ; `rel` = RIP-relative
            call    puts wrt ..plt   ; via PLT
            xor     eax, eax
            pop     rbp
            ret

            section .rodata
msg:        db  "Hello, world!", 0   ; db/dw/dd/dq instead of .byte/.word/...
```

| GAS            | NASM                |
|----------------|---------------------|
| `.text`        | `section .text`     |
| `.globl x`     | `global x`          |
| (implicit)     | `extern x`          |
| `.byte`/`.long`/`.quad` | `db`/`dd`/`dq` |
| `.string "x"`  | `db "x",0`          |
| `sym(%rip)`    | `[rel sym]`         |
| `call f@PLT`   | `call f wrt ..plt`  |

**Try it ▸**

```bash
nasm -f elf64 hello.asm -o hello.o   # -f elf64 = produce 64-bit ELF object
ld hello.o -o hello -lc --dynamic-linker /lib64/ld-linux-x86-64.so.2
```

---

## 2.3.7 From directives to the object file

Tie it together. Each directive leaves a trace in the `.o`:

```
 .globl main          ──▶ symtab entry: name=main, bind=GLOBAL
 .type main,@function ──▶ symtab entry: type=FUNC
 main: <code>         ──▶ symtab st_value = offset in .text; bytes in .text
 .size main,.-main    ──▶ symtab st_size
 call puts@PLT        ──▶ .text bytes + .rela.text reloc (R_X86_64_PLT32, puts)
 .string "..."        ──▶ bytes in .rodata
 .p2align 4           ──▶ sh_addralign + NOP/zero padding
```

**Try it ▸** Verify the trace:

```bash
gcc -c hello.c -o hello.o
readelf -sW hello.o          # symbol table: see main as FUNC GLOBAL
readelf -rW hello.o          # relocations: see the puts reloc
objdump -dr hello.o          # code + interleaved relocations
```

---

## Summary

- Compiler output is **AT&T/GAS**; documentation and NASM are **Intel**. Master
  the operand-order flip and the size suffixes/`ptr` keywords.
- An assembly file is labels + instructions + **directives**; directives are the
  control plane that shapes sections, symbols, data, and alignment.
- Section flags, symbol-binding directives, data directives, and `@`-suffixes map
  directly onto ELF section/symbol/relocation fields — directives are literally
  how you program the linker's inputs.
- `.quad symbol` and `call sym@PLT` are the everyday ways relocations get
  created.

Next: [2.4 — Calling conventions & the ABI](04-calling-conventions-abi.md)
