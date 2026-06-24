# 1.2 — Bits, Bytes, Endianness & Data Representation

Object files, ELF headers, DWARF, and instruction encodings are all just
**structured bytes**. To read them fluently you need total comfort with how
numbers are laid out in memory. This chapter makes you fluent.

---

## 1.2.1 The byte, the nibble, the word

- A **byte** is 8 bits. The fundamental addressable unit.
- A **nibble** is 4 bits = one hex digit. One byte = two hex digits (`0x00`–`0xFF`).
- "**Word**" is overloaded and platform-dependent. In x86 *historical*
  terminology: word = 16 bits, dword = 32 bits, qword = 64 bits. We'll use these
  precise names.

```
 1 byte = 8 bits:    ┌─┬─┬─┬─┬─┬─┬─┬─┐
                     │7│6│5│4│3│2│1│0│  bit indices (LSB = 0)
                     └─┴─┴─┴─┴─┴─┴─┴─┘
                       high nibble  low nibble
 0xB7 = 1011 0111  →     B           7
```

| Name in x86 ASM | Bits | Bytes | C type (LP64)         |
|-----------------|------|-------|-----------------------|
| byte            | 8    | 1     | `char`, `int8_t`      |
| word            | 16   | 2     | `short`, `int16_t`    |
| dword           | 32   | 4     | `int`, `int32_t`      |
| qword           | 64   | 8     | `long`, `int64_t`, pointer |
| oword/xmmword   | 128  | 16    | `__int128`, SSE reg   |

> **LP64 ▸** On 64-bit Linux/macOS, the data model is **LP64**: `long` and
> pointers are 64 bits, `int` stays 32 bits. Windows x64 uses **LLP64**: `long`
> is 32 bits, only `long long` and pointers are 64. This bites you when reading
> structs across platforms — and it's why ELF uses explicitly-sized types like
> `Elf64_Word` (always 32-bit) and `Elf64_Xword` (always 64-bit) instead of
> bare C types.

---

## 1.2.2 Endianness: the single most confusing byte detail

When a multi-byte integer is stored in memory, in what order do the bytes go?

Take the 32-bit value `0x0A0B0C0D`. The four bytes are `0A 0B 0C 0D`, where
`0A` is the **most significant byte** (MSB) and `0D` the **least significant**
(LSB).

```
 Value: 0x0A0B0C0D     (MSB = 0A, LSB = 0D)

 LITTLE-ENDIAN (x86-64, ARM default, RISC-V):  least-significant byte first
   addr:   +0   +1   +2   +3
         ┌────┬────┬────┬────┐
         │ 0D │ 0C │ 0B │ 0A │
         └────┴────┴────┴────┘
          LSB                MSB

 BIG-ENDIAN (network order, SPARC, s390, PowerPC BE):  MSB first
   addr:   +0   +1   +2   +3
         ┌────┬────┬────┬────┐
         │ 0A │ 0B │ 0C │ 0D │
         └────┴────┴────┴────┘
          MSB                LSB
```

Mnemonic: **little-endian = little end (least significant) comes first**.

### Why you MUST internalize this

When you hexdump an ELF file on x86-64, every multi-byte field is
**little-endian**. The ELF magic is followed by a byte (`EI_DATA`) that *tells
you* the endianness (`ELFDATA2LSB` = little, `ELFDATA2MSB` = big), and tools
must honor it.

**Try it ▸** See endianness with your own eyes:

```bash
printf '\x0d\x0c\x0b\x0a' > /tmp/le.bin
xxd /tmp/le.bin
# 00000000: 0d0c 0b0a                                ....
python3 -c "import struct;print(hex(struct.unpack('<I',open('/tmp/le.bin','rb').read())[0]))"
# 0xa0b0c0d   ← interpreted little-endian
python3 -c "import struct;print(hex(struct.unpack('>I',open('/tmp/le.bin','rb').read())[0]))"
# 0xd0c0b0a   ← interpreted big-endian
```

> **Gotcha ▸** A hex *dump* shows bytes in address order, left to right. So the
> value `0x0A0B0C0D` appears as `0d0c0b0a` in `xxd` on a little-endian box. When
> you read ELF dumps, your brain must reverse 4- and 8-byte groups.

---

## 1.2.3 Signed integers: two's complement

All mainstream hardware represents signed integers in **two's complement**. To
negate a value: invert all bits and add 1.

```
 8-bit examples:
   +5  = 0000 0101
   -5  = 1111 1011   (invert 0000 0101 → 1111 1010, +1 → 1111 1011)
   -1  = 1111 1111
   -128= 1000 0000   (most negative; no positive counterpart!)
   +127= 0111 1111
```

Key properties that affect codegen and disassembly:

- The MSB is the sign bit, but it's *not* a separate sign-magnitude bit; the
  whole encoding is one number on a wheel.
- There's one more negative value than positive (`INT_MIN` has no positive
  twin). `-INT_MIN` overflows — undefined behavior in C++.
- Addition/subtraction hardware is identical for signed and unsigned; only the
  *interpretation* of flags (CF vs OF) and the *conditional branches* differ.
  This is why the compiler emits `jb` (unsigned below) vs `jl` (signed less).

> **Sign vs zero extension ▸** When widening, `movsx`/`movsxd` (sign-extend)
> copies the sign bit leftward; `movzx` (zero-extend) fills with zeros. Picking
> the wrong one is a classic bug. C++ integer promotion rules decide which the
> compiler emits.

---

## 1.2.4 LEB128: the variable-length integer of DWARF & DWARF

DWARF (and WebAssembly, and Android DEX) encode integers in **LEB128**
(Little-Endian Base 128) to save space. You *will* decode these by hand when
reading DWARF, so learn it now.

**Unsigned LEB128 (ULEB128):** split the number into 7-bit groups, little-end
first. Set the high bit (`0x80`) on every byte *except the last*.

```
 Encode 624485 (0x9876E):
   binary: 1 0011 0001 1101 1011 1100 1110
   group into 7s from the LSB:
     0100110  0011101  1001110
       0x26     0x1d     0x4e
   little-endian order (LSB group first), set continuation bits:
     0x4e|0x80=0xCE   0x1d|0x80=0x9D   0x26 (last, no bit)=0x26
   bytes: CE 9D 26
```

**Signed LEB128 (SLEB128):** same, but sign-extend to a multiple of 7 bits and
the sign bit of the final byte (bit 6) carries the sign.

```
 ULEB128 byte:  ┌─┬─┬─┬─┬─┬─┬─┬─┐
                │C│7-bit payload│     C = continuation (1 = more bytes follow)
                └─┴─┴─┴─┴─┴─┴─┴─┘
```

**Try it ▸**

```python
def uleb128(n):
    out = bytearray()
    while True:
        b = n & 0x7f; n >>= 7
        if n: out.append(b | 0x80)
        else: out.append(b); break
    return bytes(out)
print(uleb128(624485).hex())   # ce9d26
```

> **Why it matters ▸** DWARF abbreviation codes, attribute forms
> (`DW_FORM_udata`), and the line-number program all use LEB128 heavily.
> `readelf --debug-dump` decodes it for you, but to read raw `.debug_*` bytes
> you must do it manually.

---

## 1.2.5 Alignment and padding

A type with alignment `A` must live at an address that is a multiple of `A`.
Hardware may fault or run slowly on misaligned access; the ABI mandates
alignment for performance and atomicity.

```
 struct S { char c; int i; double d; };   // x86-64 LP64

 offset:  0    1  2  3   4    5 6 7   8           15
        ┌────┬───────────┬────┬───────┬──────────────┐
        │ c  │  padding  │  i (4B)    │  d (8 bytes) │
        └────┴───────────┴────────────┴──────────────┘
          1B    3B pad      align 4       align 8
 sizeof(S) = 16 (already a multiple of 8 = alignof(S))
```

Rules:

- A struct's alignment = the max alignment of its members.
- The compiler inserts **internal padding** to align each member and **tail
  padding** so arrays of the struct stay aligned.
- `alignas`/`#pragma pack` change this; mismatched packing across translation
  units is undefined behavior and a real-world ABI break.

This matters for ELF too: section data has alignment requirements
(`sh_addralign`), and the linker pads between sections to satisfy them.

> **C++ ▸** `sizeof` includes tail padding; `[[no_unique_address]]` lets empty
> members reuse padding. The layout above is *implementation-defined* but the
> System V ABI nails it down precisely so that separately-compiled TUs agree —
> that's the whole point of an ABI.

---

## 1.2.6 Floating point (IEEE-754) at a glance

Floats are stored as sign · mantissa · 2^exponent.

```
 float (32-bit):
  31  30        23 22                              0
 ┌──┬─────────────┬─────────────────────────────────┐
 │S │  exponent   │            mantissa             │
 │1 │   8 bits    │            23 bits              │
 └──┴─────────────┴─────────────────────────────────┘
   value = (-1)^S × 1.mantissa × 2^(exponent - 127)

 double (64-bit): 1 sign, 11 exponent (bias 1023), 52 mantissa
```

Relevant facts for the toolchain:

- Float/double live in **XMM registers** and are passed/returned in XMM0–7 on
  System V.
- Float constants are emitted into `.rodata` and loaded; you'll see
  `movsd xmm0, [rip + .LC0]` with a relocation pointing into a constant pool.
- `long double` on x86-64 Linux is the 80-bit x87 extended type (`tbyte`,
  10 bytes, padded to 16) — a notorious ABI/portability hazard.

---

## 1.2.7 Reading a hex dump like a pro

Here's the start of a real 64-bit ELF, annotated. Internalize this skill — it's
how you'll verify everything in Part 4.

```
$ xxd -g1 a.out | head -4
00000000: 7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00  .ELF............
          └┬┘ └──┬──┘ │  │  │  └──────────┬─────────┘
        0x7F 'E''L''F'│  │  │      EI_PAD (zeros)
          EI_MAG0..3  │  │  └ EI_VERSION = 1
                      │  └ EI_DATA = 1 → ELFDATA2LSB (little-endian)
                      └ EI_CLASS = 2 → ELFCLASS64 (64-bit)
00000010: 03 00 3e 00 01 00 00 00 50 10 00 00 00 00 00 00  ..>.....P.......
          └─┬─┘└─┬─┘ └────┬────┘ └──────────┬──────────┘
         e_type  │     e_version       e_entry = 0x1050
        =3(DYN)  └ e_machine = 0x3E = 62 = EM_X86_64
```

Notice every multi-byte field is little-endian: `e_machine` bytes are `3e 00`
→ value `0x003e`; `e_entry` bytes `50 10 00 00 00 00 00 00` → `0x1050`.

We dissect every field of this header in [Part 4.2](../04-object-files-and-elf/02-elf-header.md).

---

## Summary

- Everything in object files is structured bytes; fluency requires comfort with
  hex, nibbles, and word sizes.
- x86-64 and ARM are **little-endian**; hex dumps show bytes in address order,
  so you must mentally reverse multi-byte fields.
- Signed integers are two's complement; sign vs zero extension is a real
  codegen decision.
- **LEB128** is the variable-length encoding pervasive in DWARF — learn to
  decode it by hand.
- Alignment and padding are dictated by the ABI and propagate into section
  layout.

Next: [1.3 — The toolchain end to end](03-toolchain-overview.md)
