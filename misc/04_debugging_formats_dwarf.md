# Debugging Formats: A Deep Dive into DWARF

This tutorial explains **why** and **how** compilers and linkers attach metadata to binaries so debuggers can reverse the compilation pipeline in your head: from **machine addresses** and **register values** back to **source files**, **lines**, **types**, **variables**, and **call stacks**.

---

## 1. Why Debugging Formats Exist

### 1.1 The core problem: mapping machine code back to source

When your toolchain turns source into an executable, it performs destructive transformations:

- Names disappear or are mangled.
- Control flow is rearranged (inlining, reordering, tail calls).
- Variables are eliminated, split, or kept only in registers briefly.
- Multiple source lines collapse into one instruction; one source line explodes into many.

A **CPU** only sees bytes and addresses. A **human debugger** wants:

- "Which file and line is PC `0x401234`?"
- "What is the type and layout of `struct foo`?"
- "Where does `argc` live *right now* (register vs stack slot)?"
- "How do I walk one frame up the stack reliably?"

A **debugging format** is a standardized way to ship **sidecar facts** in the binary (or nearby files) that stitch those worlds together.

```
  +------------------+     compile / optimize / codegen      +------------------+
  |   Source code    |  -----------------------------------> |  Machine code    |
  |   (C, C++, etc.) |                                       |  + data + relocs |
  +------------------+                                       +------------------+
         |                                                              ^
         | symbols (linker)                                             |
         v                                                              |
  +------------------+         "debug info" (optional)                  |
  | Symbol tables    |  ----------------------------------------------+
  | (.symtab, etc.)  |       maps addresses / regs -> source concepts
  +------------------+
```

**Without** debug info you might still *disassemble* and *guess* using symbols. **With** DWARF you can build **semantic** views: breakpoints on lines, pretty-printing structs, accurate backtraces across optimized code (as far as the info allows).

### 1.2 A short history

| Era / format | Rough idea | Typical container | Notes |
|--------------|------------|-------------------|--------|
| **STABS** | String-based entries embedded in symbol table aux fields or special `.stab` sections | a.out / ELF | Historically important; simple; largely superseded on modern Unix. |
| **COFF debug** | Type and line records tied to COFF sections/symbols | COFF | Windows PE lineage inherits ideas; not the modern Unix default. |
| **CodeView** | Rich record streams describing types, lines, inline info | PE (PDB often alongside) | De facto on Windows; different ecosystem from DWARF tooling. |
| **DWARF** | Tree of **DIEs** + bytecoded **line programs** + **CFI** | ELF, Mach-O, Wasm, etc. | Portable, extensible, dominant on Unix-like systems with GCC/Clang/LLVM. |

DWARF is not "the only way" to debug—platforms mix **symbol tables**, **unwind tables** (often `.eh_frame`), **note sections**, and **vendor extensions**. But DWARF is the usual answer to: *"How do I attach structured source-level metadata to an ELF object?"*

### 1.3 Comparison axes (mental model)

When comparing formats, ask:

1. **Structure**: records vs trees vs streams.
2. **Dedup**: can types be interned across TUs?
3. **Relocation friendliness**: can debug sections live in relocatable objects cleanly?
4. **Optimization tolerance**: line tables + location lists + `DW_TAG_inlined_subroutine`.
5. **Split builds**: can debug info be separated from the linked binary?

DWARF scores well on (2), (3), (4), and (5) with DWARF 5 **Split DWARF**.

---

## 2. DWARF Overview

### 2.1 What DWARF stands for

**DWARF** is a pun on **ELF**: "ELF" and "DWARF" were named together. Officially it is *not* an acronym today, though you'll sometimes see *"Debugging With Attributed Record Formats"* treated informally as a backronym.

### 2.2 Versions (DWARF 1–5)

Very high-level evolution:

| Version | Themes |
|---------|--------|
| **DWARF 1** | Older, less uniform; largely historical. |
| **DWARF 2** | Major redesign: DIE tree model, line programs, separate sections; widely supported baseline. |
| **DWARF 3** | Incremental extensions; more attributes/forms; improved descriptions. |
| **DWARF 4** | **Type units** (`TU`) in `.debug_types`, new line-table features, compressed representation options. |
| **DWARF 5** | **Split DWARF** (`.dwo`), header/section layout refinements (`debug_str_offsets`, `debug_line_str`, etc.), `debug_loclists`/`debug_rnglists` in many producers, performance + tooling-oriented cleanup. |

Toolchains commonly emit **DWARF 4** or **DWARF 5** today; consumers (GDB, LLDB, `llvm-symbolizer`) implement subsets with version-specific fallbacks.

### 2.3 Design principles

1. **Separate debug sections** from normal code/data so **stripping** (`strip`, `--strip-debug`) is well-defined.
2. **Tree-structured semantics** (DIEs) for types, scopes, and entities.
3. **Bytecode programs** for **line mapping** and **location expressions**—compact and reloc-friendly compared to naive per-PC records.
4. **Extensible attribute encodings** via **forms** and vendor tags.
5. **Relocation** at link time: many DIE references are **offsets within debug sections** or use forms that patch cleanly.

### 2.4 Conceptual bridge: source ⭤ DWARF ⭤ binary

```
  +-----------+    lexical scope / types / globals
  |  Source   |
  +-----------+
        |
        |  Front-end + semantic analysis
        v
  +-----------------------------+
  |   DIE tree (per compile     |
  |   unit + referenced TUs)    |
  |   describes *meaning*       |
  +-----------------------------+
        |        \
        |         \  Line Number Program
        |          v
        |      +-------------------------+
        |      | Line table: addr ->     |
        |      | file:line:column        |
        |      +-------------------------+
        |
        |  + CFI (often .eh_frame too)
        v
  +-----------------------------+
  | Addresses in the text (PC)  |
  | + stack slots / registers   |
  +-----------------------------+
```

---

## 3. DWARF Sections in ELF (classical layout)

> Names below follow common **ELF + DWARF** usage. Exact section names can vary slightly by linker/compiler (e.g. `zdebug` compressed sections), but the roles stay recognizable.

### 3.1 `.debug_info` — the main debugging information

Contains a forest of **DIE trees**, one root typically being `DW_TAG_compile_unit` per translation unit (TU). Almost everything you think of as "types and variables" lives here as interconnected nodes.

### 3.2 `.debug_abbrev` — abbreviation tables

`debug_info` would bloat if every DIE repeated full metadata about its **tag** and **attributes**.

**Abbreviations** encode a template: *this DIE shape has these attributes with these forms*. The `debug_info` entry then mostly stores **attribute values** plus an abbreviation code.

Benefit: huge compression + faster parsing.

### 3.3 `.debug_line` — line number information

Encodes a **Line Number Program** (state machine bytecode) that generates a matrix mapping **PC ranges** to **file / line / column / discriminator** (and flags like `is_stmt`, `prologue_end`, etc.).

### 3.4 `.debug_str` — string table

Holds nul-terminated strings **referenced indirectly** from attributes using `DW_FORM_strp` (and related mechanisms). Centralizing strings deduplicates repeated names.

DWARF 5 adds companion pieces like **string offsets tables** (see §8).

### 3.5 `.debug_aranges` — address range index (optional, but common)

A fast **lookup table**: *"which compile unit covers address A?"* Useful for quickly locating the relevant slice of `debug_info` / line tables.

### 3.6 `.debug_frame` — Call Frame Information (CFI)

Describes how to **unwind** registers at any PC: where saved return address is, CFA rule, callee-saved regs.

Note: Many systems also use **`.eh_frame`** for exception handling; unwinders often can consume DWARF CFI from either. Debuggers need consistent frame description to walk stacks.

### 3.7 `.debug_loc` — location lists (classic; DWARF ≤4 style)

For entities whose storage changes during execution (register vs spilled), a single static `DW_AT_location` isn't enough. **Location lists** map **PC ranges** → **DWARF expression**.

DWARF 5 commonly moves this role to **`.debug_loclists`** (new header/versioning model). Tools still say "loclists" generically.

### 3.8 `.debug_ranges` — non-contiguous address ranges (classic)

Attributes like `DW_AT_ranges` may point to range lists when an object code region is non-contiguous (cold blocks, linker reordering, etc.).

DWARF 5 analog: **`.debug_rnglists`**.

### 3.9 `.debug_types` — type units (DWARF 4)

Enables splitting **type graphs** into **Type Units (TUs)** with **signatures**, so the linker can **deduplicate identical types** across object files.

DWARF 5 continues these ideas with refined unit models; producers vary in exactly which units land where.

### 3.10 `.debug_macro` — macro information

If compiled with macro debug options, encodes macro definitions / undefs and their source associations so debuggers can expand macros sensibly.

### 3.11 Diagram: how sections reference each other

```
                    +----------------+
                    |  .debug_abbrev |
                    +----------------+
                           ^
                           | abbrev table # used by DIE encodings
                           |
+----------------+    +----------------+    +----------------+
| .debug_aranges |--->|  .debug_info   |<---|   .debug_str   |
+----------------+    |  (DIE graph)   |    +----------------+
      (PC -> CU)      +--------+-------+      ^  (names via
                           |    |             |   strp forms)
            ranges /       |    |             |
            loclists       |    +-------------+
                           |                  |
                   +-------v--------+   +-----v----------+
                   | .debug_ranges  |   | .debug_loc(*)  |
                   | or .debug_     |   |(.debug_loclists|
                   |     rnglists   |   | in DWARF5)     |
                   +----------------+   +----------------+
                           ^
                           |
                      +----+-----+
                      | .text    |
                      | (code)   |
                      +----+-----+
                           ^
                           |
                      +----+-----+
                      |.debug_line|
                      |(LN prog)  |
                      +-----------+

   CFI (unwind rules, keyed by PC):
        +----------------+       sometimes mirrored / duplicated
        | .debug_frame   |       conceptually alongside code
        +----------------+

   (*) Classic name `.debug_loc`; DWARF 5 may use `.debug_loclists`.
```

**Key idea**: `.text` gives *instructions at PC*, while DWARF sections describe *what that PC means* in source and *how to find variable homes* as PC changes.

---

## 4. DIE (Debugging Information Entry) Tree

### 4.1 What a DIE is

A **DIE** is a node in the debug-info graph:

- A **tag** (`DW_TAG_*`) says what the node *means* (function, type, variable…).
- A list of **attributes** (`DW_AT_*`) carries properties (name, type reference, linkage name, etc.).
- Each attribute is encoded with a **form** (`DW_FORM_*`) describing how the value is represented (immediate, string offset, block, reference…).

Children DIEs nest to represent scopes:

```
DW_TAG_compile_unit
  +-- DW_TAG_namespace (optional)
  |     +-- DW_TAG_structure_type
  |           +-- DW_TAG_member ...
  +-- DW_TAG_subprogram  (function)
        +-- DW_TAG_variable (local)
        +-- DW_TAG_lexical_block
              +-- DW_TAG_variable
```

### 4.2 Common tag types (not exhaustive)

| Tag | Typical meaning |
|-----|------------------|
| `DW_TAG_compile_unit` | Root for one TU: compiler name, language, line table pointer, base addresses. |
| `DW_TAG_subprogram` | Function / method / outline compiler artifact. |
| `DW_TAG_inlined_subroutine` | Inlined call site; carries `abstract_origin` chains. |
| `DW_TAG_variable` | Variable (local/global/static depending on attributes/location). |
| `DW_TAG_formal_parameter` | Parameter. |
| `DW_TAG_base_type` | Fundamental type: `int`, `float`, etc. |
| `DW_TAG_pointer_type`, `DW_TAG_reference_type`, `DW_TAG_rvalue_reference_type` | Indirection types. |
| `DW_TAG_typedef` | Name alias. |
| `DW_TAG_structure_type`, `DW_TAG_class_type`, `DW_TAG_union_type`, `DW_TAG_enumeration_type` | Aggregates / enums. |
| `DW_TAG_member` | Field, offset, accessibility. |
| `DW_TAG_array_type` | Array with subranges. |
| `DW_TAG_subroutine_type` | Function types. |
| `DW_TAG_lexical_block`, `DW_TAG_try_block`, `DW_TAG_catch_block` | Source-like scope nesting. |
| `DW_TAG_label` | Named label (when emitted). |

### 4.3 Important attributes (not exhaustive)

| Attribute | Role |
|----------|------|
| `DW_AT_name` | Human-readable name. |
| `DW_AT_linkage_name` | Mangled linkage name (C++ ABI). |
| `DW_AT_decl_file`, `DW_AT_decl_line`, `DW_AT_decl_column` | Source declaration point. |
| `DW_AT_type` | Points to another DIE describing the type. |
| `DW_AT_low_pc`, `DW_AT_high_pc` | Contiguous address span for the entity (often functions). |
| `DW_AT_ranges` | Non-contiguous ranges. |
| `DW_AT_location`, `DW_AT_frame_base` | **DWARF expressions** / location lists describing where data lives. |
| `DW_AT_external` | Visibility across TUs. |
| `DW_AT_const_value` | Constant value for enums / constexpr-like cases. |
| `DW_AT_byte_size`, `DW_AT_bit_size`, `DW_AT_data_bit_offset` | Layout. |
| `DW_AT_sibling` | Optional acceleration for tree walkers (skip subtree). |
| `DW_AT_specification`, `DW_AT_abstract_origin` | Deduplication / out-of-line vs inline chaining. |

### 4.4 Attribute forms (`DW_FORM_*`)

Examples:

| Form family | Meaning |
|------------|---------|
| `DW_FORM_addr` | Raw machine address (needs relocation). |
| `DW_FORM_ref1/2/4/8`, `DW_FORM_ref_addr`, `DW_FORM_ref_sig8` | References to other DIEs or type signatures (variant details differ by version). |
| `DW_FORM_strp` | Offset into `.debug_str`. |
| `DW_FORM_string` | Inline string (no pooling). |
| `DW_FORM_block1/2/4`, `DW_FORM_exprloc` | Byte blocks; **exprloc** is a DWARF expression for locations. |
| `DW_FORM_udata/sdata` | LEB128 integers. |
| `DW_FORM_flag_present` | Boolean attribute present/absent. |

**Reference vs address**: a `ref` is **within the debug-info address space** (offset-based). An `addr` is a **program address** in the loaded process.

### 4.5 Sample C program → DIE sketch

Source:

```c
static int counter;

int add(int a, int b) {
    int sum = a + b;
    counter += sum;
    return sum;
}
```

ASCII sketch (simplified; real DIEs include many more attributes):

```
DW_TAG_compile_unit
|  DW_AT_name        -> "example.c"
|  DW_AT_language    -> DW_LANG_C
|  DW_AT_comp_dir     -> "/home/user/proj"
|  DW_AT_low_pc       -> 0x401000  (after link/load)
|  DW_AT_high_pc      -> ...        (or ranges)
|  DW_AT_stmt_list    -> offset into .debug_line
|
|-- DW_TAG_variable  ("counter")
|      DW_AT_name -> "counter"
|      DW_AT_linkage_name -> ... (maybe)
|      DW_AT_external -> false
|      DW_AT_type -> {DW_TAG_base_type "int"}
|      DW_AT_location -> DW_OP_addr 0x404010  (static storage; exact encoding varies)
|
\-- DW_TAG_subprogram  ("add")
       DW_AT_name -> "add"
       DW_AT_low_pc / DW_AT_high_pc -> function extents
       DW_AT_type -> {DW_TAG_base_type "int"}
       |
       |-- DW_TAG_formal_parameter ("a")
       |      DW_AT_type -> {int}
       |      DW_AT_location -> expr: DW_OP_reg5  (example only)
       |
       |-- DW_TAG_formal_parameter ("b")
       |      DW_AT_type -> {int}
       |      DW_AT_location -> expr: DW_OP_reg4  (example only)
       |
       \-- DW_TAG_variable ("sum")
              DW_AT_type -> {int}
              DW_AT_location -> expr: DW_OP_fbreg -12  (example: FP-relative)
```

This tree is how GDB finds `sum`, knows it is an `int`, and evaluates a **location description** at the current PC.

---

## 5. Line Number Program

### 5.1 Why a bytecode program?

Naively storing `(PC → line)` for every instruction explodes in size. Instead DWARF stores a compact **program** that updates a *running state* as PC advances.

### 5.2 The line number state machine (core registers)

Think of an interpreter with these key fields (names per DWARF conceptually):

```
+--------------------------------------------------------------+
|  Line Number Program State (conceptual)                      |
+----------+---------------------------------------------------+
| address  | 'current PC' while stepping through code addresses|
| file     | index into file table                             |
| line     | source line number                                |
| column   | source column                                     |
| is_stmt  | recommended breakpoint location?                  |
| basic_   | start of basic block?                             |
| end_     | end of function sequence?                         |
| ...      | prologue_end, epilogue_begin, discriminator, ISA  |
+----------+---------------------------------------------------+
```

The program emits **rows** into a **line matrix** when state is committed (depending on opcode semantics).

### 5.3 Opcodes (families)

DWARF line programs mix:

1. **Standard opcodes** (`DW_LNS_*`): e.g. advance line, advance PC, set file, set column, copy (emit row).
2. **Extended opcodes** (`DW_LNE_*`): variable-length instructions; e.g. `end_sequence`, `set_address`, define file in extended forms (details version-dependent).
3. **Special opcodes**: pack **line delta + address advance** into one byte using header parameters:

```
  opcode_base, line_base, line_range, min_instr_length

  special = opcode - opcode_base
  address_inc = (special / line_range) * min_instr_length
  line_inc    = line_base + (special % line_range)
```

**Intuition**: special opcodes are the big win for density—most matrix rows look like small deltas.

### 5.4 Line matrix mental model

```
   PC / vma    File        Line   Col   Flags
  ----------  ----------   ----   ----  -------
  0x401000    example.c     10      0   is_stmt
  0x401004    example.c     11      5
  0x40100c    example.c     12      9   prologue_end
     ...
```

Debugger query: given **PC**, binary search the matrix (or use CU aranges → line program slice) → **source coordinate**.

### 5.5 ASCII diagram: state machine stepping

```
    START
      |
      v
+-------------+     read byte      +-------------------------+
| Fetch opcode|------------------->| Decode: std/ext/special |
+-------------+                    +------------+------------+
      ^                                      |
      |                                      v
      |                            +---------v---------+
      |                            | Update state regs |
      |                            | maybe emit row    |
      |                            +---------+---------+
      |                                      |
      | end_sequence?                        |
      |                              no      |
      +--------------------------------------+
                 |
                yes
                 v
              +------+
              | DONE |
              +------+
```

### 5.6 “Special”, “discriminator”, and optimized code

When statements are merged or reordered, a **discriminator** helps distinguish multiple distinct inlined or merged regions mapping to the same line. Without it, setting a breakpoint on a line becomes ambiguous.

---

## 6. Location Descriptions

### 6.1 The problem

A variable might:

- Start in a register,
- Get spilled to `[rsp+0x20]`,
- Be **split** across SSA-like pieces,
- Exist only in a SIMD register temporarily,

…all within one function. `DW_AT_location` therefore often points to a **location list**: *(PC range → expr)*.

### 6.2 DWARF expressions: a stack machine

A **DWARF expression** is a byte sequence of `DW_OP_*` operations manipulating a stack of values **during evaluation by the debugger** (not executed by CPU):

```
  Expression bytes:  DW_OP_fbreg  DW_OP_deref
                       (example conceptual pattern)

  Debugger maintains:
     stack: [ ... values ... ]
     context: current thread registers, memory read, CFA from CFI, etc.
```

Evaluation yields either:

- A **location**: address, register number, implicit value, literal piecewise description, etc.
- Or failure / undefined during prologue gaps if info is missing.

### 6.3 Some `DW_OP_*` examples (illustrative)

| Operation | Meaning (simplified) |
|-----------|----------------------|
| `DW_OP_addr` | Push absolute address (relocated). |
| `DW_OP_regN` | Value is in register N. |
| `DW_OP_bregN` | Push `register N + sleb128 offset`. |
| `DW_OP_fbreg` | Offset relative to **frame base** from CFI (`DW_AT_frame_base`). |
| `DW_OP_STACK_VALUE` | Top of stack is a **value**, not an address. |
| `DW_OP_piece`, `DW_OP_bit_piece` | Compose aggregate from fragments. |
| `DW_OP_deref` | Treat top as address; load from memory. |
| `DW_OP_plus`, `DW_OP_minus`, … | Arithmetic. |
| `DW_OP_entry_value(...)` (newer) | Expression evaluated in caller frame; crucial for optimized debugging. |

Exact availability depends on DWARF version and producer quality.

### 6.4 Location lists

```
PC in [0x401000, 0x401024):   expr A  (e.g., DW_OP_reg3)
PC in [0x401024, 0x401060):   expr B  (e.g., DW_OP_breg7 +0 + DW_OP_deref-like chain)
```

Debugger algorithm:

1. Find active range for current PC.
2. Evaluate corresponding expression using registers/memory.
3. If missing range → variable is **optimized out** / unavailable.

### 6.5 “Complex expression” story (readable)

Consider reading a `struct` field:

- Debugger starts at variable location (address or register holding pointer).
- Adds member offset from `DW_TAG_member` `DW_AT_data_member_location` (often `DW_FORM_sdata` constant offset or another expression).

The *type graph* + *location expr* compose into a usable address.

---

## 7. Call Frame Information (CFI)

### 7.1 Why CFI exists

To print a backtrace, the debugger must answer, for each frame:

- Where is the **return address**?
- Where is the **previous frame pointer / CFA**?
- How do I recover callee-saved registers?

**CFI** is PC-indexed unwind rules. It exists independent of DWARF *types*, but DWARF consumers rely on it heavily.

### 7.2 CIE and FDE

CFI is stored as records:

```
+-------------------------------------------------------------------+
|  CIE  (Common Information Entry)                                  |
|   - augmentation string ("eh" etc.)                               |
|   - code alignment factor, data alignment factor                  |
|   - return address column                                         |
|   - initial instructions (defaults)                               |
+----------------------------+--------------------------------------+
                             |
           +-----------------+-----------------+
           |                                   |
           v                                   v
+---------------------------+      +----------------------------+
| FDE (Frame Description    |      | FDE                        |
|      Entry) for function  | ...  |                            |
|  - PC range               |      |  - PC range                |
|  - CIE pointer            |      |  - CIE pointer             |
|  - CI instructions        |      |  - CI instructions         |
+---------------------------+      +----------------------------+
```

- **CIE** shares constants across many **FDEs**.
- Each **FDE** covers a **PC range** in `.text`.

### 7.3 CFI instructions (DW_CFA_*)

Examples (simplified):

| Instruction (concept) | Effect |
|-----------------------|--------|
| `DW_CFA_def_cfa_register` | CFA = register + offset rule begins with reg as base. |
| `DW_CFA_def_cfa_offset` | CFA = old_SP + offset. |
| `DW_CFA_offset` | Saved reg at `CFA + offset`. |
| `DW_CFA_remember_state` / `DW_CFA_restore_state` | Save/restore unwind rule state across epilogue/prologue subranges. |
| `DW_CFA_advance_loc` | Rules apply after advancing PC delta. |

Because prologue/epilogue instructions mutate the stack pointer stepwise, CFI often **updates across small PC slices** within one function.

### 7.4 How a debugger unwinds (conceptual algorithm)

Given current **RSP/RBP**, **PC**, and register set:

```
Frame 0 (innermost):
  1) Locate FDE covering PC
  2) Execute CFI program up to PC -> compute CFA, find return addr slot
  3) Read previous PC & registers using memory/reg rules

Frame 1:
  repeat using previous CFA as anchor

... until bottom or unwind failure
```

### 7.5 ASCII diagram: stack unwinding

```
High addresses
 |
 |   +-----------------------------+
 |   |  Frame 1 locals ...         |
 |   +-----------------------------+
 |   |  saved regs / pads          |
 |   +-----------------------------+ <-- CFA for Frame 1
 |   |  return address  ------->   |----> used to get caller PC
 |   +-----------------------------+
 |   |  Frame 0 locals ...         |
 |   +-----------------------------+
 |   |  saved regs                 |
 |   +-----------------------------+ <-- CFA for Frame 0
 v   |  return address             |
     +-----------------------------+
Low addresses

CFI answers: "given PC in Frame 0's prologue/epilogue/body,
             where is my CFA now, and where is the saved RA?"
```

If CFI and actual code disagree (rare bug/mismatch), backtraces become garbage—**"stack corrupted"** symptoms sometimes are just **bad unwind metadata**.

---

## 8. DWARF 5 Improvements (selected)

### 8.1 Split DWARF: `.dwo` + skeleton

**Problem**: Large projects embed enormous debug sections in every `.o` and in final linked images.

**Split DWARF** stores the bulk of debug data in **per-compilation `.dwo`** objects; the linked executable keeps a compact **skeleton** that references them.

```
Traditional:
   a.out contains .debug_info (huge)

Split:
   a.out contains skeleton debug sections (small)
   + /path/to/foo.dwo contains the heavy DIEs/line/macros for foo.c's CU
```

Benefits:

- Faster I/O for non-debug workflows.
- Sharing `.dwo` across distributed build caches.
- Smaller symbols servers payloads when combined with proper tooling.

### 8.2 `.debug_str_offsets`

Speed-parses string-heavy tables by providing parallel **offsets** into `.debug_str` in a structured way; reduces per-attribute string parsing costs in some consumers.

### 8.3 Line number header changes

DWARF 5 modernizes line program headers (directory/file representation, optional new fields like **MD5** checksums for files when present, etc.—exact feature set depends on producer). Goal: more robust **source file identity** and tooling performance.

### 8.4 Loclists / rnglists sectioning

DWARF 5 formalizes **loclists** and **rnglists** with unit headers that are easier to validate and relocate; consumers migrate from older `.debug_loc`/`.debug_ranges` layouts.

### 8.5 Performance themes

- Better factoring + tables for quicker random access.
- More **indexed** data (`debug_aranges`, accelerator tables in newer proposals/usages depending on toolchain).
- Continued push for **reducing linker work** and **duplicate type graphs**.

---

## 9. Practical Examples

### 9.1 Compile with debug info

Typical GCC/Clang flags:

```bash
# Classic DWARF embedded in object files
cc -g -O0 -o prog main.c other.c

# Request a specific DWARF version (toolchain-dependent defaults vary)
cc -gdwarf-5 -g -O0 -o prog main.c

# Split DWARF: per-TU .dwo + skeleton in linked binary
cc -gsplit-dwarf -g -O0 -c main.c -o main.o
cc -gsplit-dwarf -g -O0 -o prog main.o other.o
```

Also common:

```bash
# Preserve macro info (GCC/Clang; exact spelling may vary)
cc -g3 ...
```

### 9.2 Inspect DWARF with standard tools

**GNU `readelf`** — section presence / high-level:

```bash
readelf --debug-dump=info a.out | less
readelf -S a.out | rg debug
```

**`llvm-dwarfdump`** (LLVM; very readable summaries):

```bash
llvm-dwarfdump --all a.out | less
llvm-dwarfdump --name=add a.out
```

**GNU `dwarfdump`** (name collision across distros; often from `libdwarf` tooling packages):

```bash
dwarfdump -i a.out | less
```

When using split DWARF, point tools at the executable; many tools follow `.dwo` references automatically if paths embed correctly.

### 9.3 How GDB uses DWARF internally (conceptual)

GDB (simplified pipeline):

1. **Symtab expansion**: loads symbols + DWARF index slices on demand for a CU.
2. **Line tables**: PC ↔ source coordinates for `list`, stepping, breakpoints by line.
3. **DIE graph**: resolves variable names in scope; follows `DW_AT_type` to pretty-print.
4. **Location lists + expr eval**: reads registers/memory to fetch value, or marks optimized out.
5. **CFI**: unwinds stack for `bt`, `frame`, `up/down`.

Optimized code path: GDB leans more on **DWARF location quality**; `DW_OP_entry_value` and friends reduce "value optimized out" pain when producers emit them.

### 9.4 Optimizations vs `-g` (`-O2 -g`)

`-g` does **not** mean "debug-friendly execution." **Optimization** can:

- Inline functions (`DW_TAG_inlined_subroutine` appears).
- Eliminate variables; location lists have holes.
- Reorder lines; line tables remain but look "jumping."
- Clone loops; addresses may correspond oddly to source.

Rule of thumb:

- **`-O0 -g`**: closest mapping, easiest introspection.
- **`-O2 -g`**: realistic production-like debugging; quality depends on **Clang/GCC version** and **DWARF features emitted**.

Try this experiment:

```bash
cc -O2 -g -gdwarf-5 -o prog_O2 main.c
cc -O0 -g -gdwarf-5 -o prog_O0 main.c
llvm-dwarfdump --name=your_function prog_O2 | less
llvm-dwarfdump --name=your_function prog_O0 | less
```

Compare `DW_TAG_inlined_subroutine` presence, `DW_AT_location` complexity, and line table density.

---

## 10. Further reading and mental checklist

When reading DWARF in the wild, ask:

1. **Which compile unit** covers this PC? (`.debug_aranges` / CU low/high PC)
2. **Which line**? (`.debug_line` matrix row)
3. **Which DIE** names this entity and its `DW_AT_type` chain?
4. **Which location list entry** applies at this PC?
5. **Which FDE** explains unwinding at this PC?

Mastering those five questions maps cleanly onto what GDB/LLDB do—just with caching, clever indexes, and years of bugfixes beneath the hood.

---

## Appendix A — Glossary (quick)

| Term | Meaning |
|------|---------|
| **DIE** | Debugging Information Entry; DWARF tree node. |
| **TU** | Translation unit; one compiled source after `#include` expansion (conceptually). |
| **CU** | Compile unit DIE root in `debug_info`. |
| **CFI** | Call Frame Information; unwind rules. |
| **CIE/FDE** | Common Information Entry / Frame Description Entry. |
| **CFA** | Canonical Frame Address; reference stack pointer for unwind offsets. |
| **vma** | Virtual memory address (loading may add slide). |
| **Split DWARF** | Debug bulk stored in `.dwo` with skeleton in main binary. |

---

*End of tutorial.*
