# 4.2 — The ELF Header, Byte by Byte

The ELF header is a 64-byte structure (for ELF64) at file offset 0. It is the
root of the whole file: it identifies the file and points to the two header
tables. We'll decode it field by field against a real binary.

---

## 4.2.1 The `Elf64_Ehdr` structure

```c
#define EI_NIDENT 16
typedef struct {
    unsigned char e_ident[EI_NIDENT]; /*  0  identification bytes        */
    Elf64_Half    e_type;             /* 16  object file type            */
    Elf64_Half    e_machine;          /* 18  target architecture         */
    Elf64_Word    e_version;          /* 20  ELF version (1)             */
    Elf64_Addr    e_entry;            /* 24  entry point virtual address */
    Elf64_Off     e_phoff;            /* 32  program header table offset */
    Elf64_Off     e_shoff;            /* 40  section header table offset */
    Elf64_Word    e_flags;            /* 48  processor-specific flags    */
    Elf64_Half    e_ehsize;           /* 52  this header's size (64)     */
    Elf64_Half    e_phentsize;        /* 54  size of one program header  */
    Elf64_Half    e_phnum;            /* 56  number of program headers   */
    Elf64_Half    e_shentsize;        /* 58  size of one section header  */
    Elf64_Half    e_shnum;            /* 60  number of section headers    */
    Elf64_Half    e_shstrndx;         /* 62  section-name strtab index   */
} Elf64_Ehdr;                         /* 64 bytes total                  */
```

```
   Offset map (64 bytes):
   0                                              15 16  18  20      24
   ┌────────────────────────────────────────────────┬───┬───┬───────┬──────...
   │              e_ident[16]                       │typ│mac│version│e_entry
   └────────────────────────────────────────────────┴───┴───┴───────┴──────...
   24      32      40      48   52  54  56  58  60  62
   ...entry │e_phoff│e_shoff│flgs│ehs│phs│phn│shs│shn│shstrndx
```

---

## 4.2.2 The `e_ident` array (the first 16 bytes)

These 16 bytes are readable without knowing endianness or class — they bootstrap
the rest of the parse.

```
   byte:  0    1    2    3    4       5       6        7         8..15
        ┌────┬────┬────┬────┬───────┬───────┬────────┬─────────┬──────────┐
        │0x7F│ 'E'│ 'L'│ 'F'│ CLASS │ DATA  │VERSION │ OSABI   │ABIVERSION│
        │    │    │    │    │       │       │        │         │ + padding│
        └────┴────┴────┴────┴───────┴───────┴────────┴─────────┴──────────┘
          └──── magic ────┘
```

| Index | Name           | Values                                                |
|-------|----------------|-------------------------------------------------------|
| 0–3   | `EI_MAG0..3`   | `0x7F 'E' 'L' 'F'` — the magic number                 |
| 4     | `EI_CLASS`     | `1`=ELFCLASS32, `2`=ELFCLASS64                        |
| 5     | `EI_DATA`      | `1`=ELFDATA2LSB (little), `2`=ELFDATA2MSB (big)       |
| 6     | `EI_VERSION`   | `1` (EV_CURRENT)                                      |
| 7     | `EI_OSABI`     | `0`=System V, `3`=Linux, `9`=FreeBSD, …               |
| 8     | `EI_ABIVERSION`| usually `0`                                           |
| 9–15  | `EI_PAD`       | reserved, zero                                        |

> **Why magic + class + data come first ▸** A tool must read `EI_CLASS` and
> `EI_DATA` *before* it can interpret any multi-byte field, because they
> determine struct size (32 vs 64) and endianness. That's why they're plain
> bytes at fixed positions, ahead of everything else.

> **OSABI nuance ▸** Plain Linux executables usually carry `EI_OSABI = 0`
> (System V), not 3. The `3` (GNU/Linux) appears when GNU-specific features
> (like STT_GNU_IFUNC or certain symbol bindings) are used. Don't be surprised by
> `0` on Linux.

---

## 4.2.3 The remaining fields, decoded

### `e_type` (offset 16) — file type
```
   1 ET_REL   relocatable (.o)
   2 ET_EXEC  fixed-address executable (non-PIE)
   3 ET_DYN   shared object OR PIE executable
   4 ET_CORE  core dump
   0xFE00-0xFEFF OS-specific; 0xFF00-0xFFFF processor-specific
```

### `e_machine` (offset 18) — architecture
```
   0x3E  62  EM_X86_64       0xB7 183 EM_AARCH64
   0x03   3  EM_386          0xF3 243 EM_RISCV
   0x28  40  EM_ARM          0x14  20 EM_PPC
```

### `e_entry` (offset 24) — entry point
The virtual address where execution begins **after** the loader finishes — this
is `_start` (from `crt1.o`), *not* `main`. For a shared library it may be 0. For
PIE (`ET_DYN`) it's a *file-relative* value to which the load base is added.

### `e_phoff` / `e_shoff` (offsets 32 / 40)
File offsets of the program header table and section header table. In a typical
executable, program headers immediately follow the ELF header (`e_phoff = 64`),
and section headers sit at the very end (`e_shoff` large). `e_shoff = 0` means no
section header table (a stripped-to-the-bone binary).

### `e_flags` (offset 48)
Processor-specific. Zero on x86-64. On ARM/MIPS/RISC-V it encodes ABI variants
(e.g. RISC-V float ABI, ARM EABI version) — mismatches here cause linker errors
like "uses VFP register arguments, output does not".

### Size/count fields (offsets 52–62)
```
   e_ehsize    = 64   (sizeof Elf64_Ehdr)
   e_phentsize = 56   (sizeof Elf64_Phdr)   e_phnum = number of segments
   e_shentsize = 64   (sizeof Elf64_Shdr)   e_shnum = number of sections
   e_shstrndx  = index into the section header table of the .shstrtab section
```

> **`e_shstrndx` ▸** Section *names* are strings stored in a section called
> `.shstrtab`. To print section names, a tool reads section header
> `[e_shstrndx]`, finds that string table, then for each section uses its
> `sh_name` as an offset into it. Chicken-and-egg solved by this one index.

> **The escape hatch for huge files ▸** `e_phnum`/`e_shnum` are 16-bit. If a
> file has ≥ 0xFF00 sections, `e_shnum = 0` and the real count lives in
> `sh_size` of section header 0 (the otherwise-unused null section). Similarly
> `e_phnum = PN_XNUM (0xFFFF)` defers to `sh_info` of section 0. Rare but real.

---

## 4.2.4 Full annotated hex dump

Here is a real `ET_DYN` (PIE) header. Read every byte:

```
$ xxd -g1 -l 64 demo
00000000: 7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
          │  E  L  F  │  │  │  │  └─── EI_PAD (zero) ───┘
          magic       │  │  │  └ EI_OSABI=0 (System V)
                      │  │  └ EI_VERSION=1
                      │  └ EI_DATA=1 (little-endian)
                      └ EI_CLASS=2 (ELF64)

00000010: 03 00 3e 00 01 00 00 00 50 10 00 00 00 00 00 00
          └─┬─┘ └─┬─┘ └────┬────┘ └───── e_entry ───────┘
        e_type  e_machine  e_version       = 0x1050
        =3 DYN  =0x3e X86_64   =1

00000020: 40 00 00 00 00 00 00 00 d8 39 00 00 00 00 00 00
          └────── e_phoff = 0x40 ──────┘ └─ e_shoff = 0x39d8 ─┘
          (program headers right after the ELF header)

00000030: 00 00 00 00 40 00 38 00 0d 00 40 00 1f 00 1e 00
          └─e_flags─┘ │   │   │   │   │   │   │   └ e_shstrndx=0x1e (30)
                      │   │   │   │   │   │   └ e_shnum = 0x1f (31)
                      │   │   │   │   │   └ e_shentsize = 0x40 (64)
                      │   │   │   │   └ e_phnum = 0x0d (13)
                      │   │   │   └ e_phentsize = 0x38 (56)
                      │   │   └ (high bytes of phentsize)
                      │   └ e_ehsize = 0x40 (64)
                      └ (e_ehsize low byte)
```

Cross-check with the tool:

```bash
readelf -h demo
# ELF Header:
#   Magic:   7f 45 4c 46 02 01 01 00 ...
#   Class:                             ELF64
#   Data:                              2's complement, little endian
#   Type:                              DYN (Position-Independent Executable file)
#   Machine:                           Advanced Micro Devices X86-64
#   Entry point address:               0x1050
#   Start of program headers:          64 (bytes into file)
#   Start of section headers:          14808 (bytes into file)
#   Number of program headers:         13
#   Number of section headers:         31
#   Section header string table index: 30
```

Every number matches the bytes you decoded. This is the payoff of §1.2:
little-endian fields read backward in the dump.

---

## 4.2.5 Validating an ELF header (what loaders check)

When `execve` or `ld.so` opens a file, it sanity-checks the header before
trusting anything:

```
   1. e_ident[0..3] == 7F 45 4C 46         (magic)
   2. EI_CLASS matches the machine (ELF64 on a 64-bit kernel)
   3. EI_DATA matches host endianness
   4. e_machine is supported by this CPU
   5. e_type is ET_EXEC or ET_DYN (you can't execve a .o)
   6. e_version == 1
   7. e_phoff + e_phnum*e_phentsize within the file (bounds)
```

A corrupt or mismatched header is why you get `cannot execute binary file: Exec
format error` (`ENOEXEC`) — the kernel rejected step 1–5.

**Try it ▸** Break the magic and watch it fail:

```bash
cp demo broken && printf '\x00' | dd of=broken bs=1 seek=0 count=1 conv=notrunc 2>/dev/null
./broken; echo "exit=$?"          # bash: cannot execute / Exec format error
file broken                       # "data" — no longer recognized as ELF
```

---

## Summary

- The ELF header is 64 bytes at offset 0 and is the root pointing to both header
  tables.
- `e_ident`'s first bytes (magic, `EI_CLASS`, `EI_DATA`) are position-fixed and
  endianness-free so tools can bootstrap parsing.
- `e_type` distinguishes REL/EXEC/DYN/CORE (PIE is DYN); `e_machine` the arch;
  `e_entry` is `_start`, not `main`.
- `e_phoff`/`e_shoff` locate the program/section header tables; `e_shstrndx`
  locates the section-name string table.
- You can now hand-decode a header from a hex dump and cross-check with
  `readelf -h`.

Next: [4.3 — Sections & the section header table](03-sections.md)
