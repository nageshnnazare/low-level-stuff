# 6.2 — DIEs: The Debug Information Tree

The heart of DWARF is `.debug_info`: a tree of **DIEs** (Debugging Information
Entries) describing every function, variable, type, and scope in your program.
This chapter explains DIEs, attributes, the abbreviation mechanism, and how to
read the tree.

---

## 6.2.1 The DIE: DWARF's universal node

A **DIE** describes one program entity. Every DIE has:
- A **tag** (`DW_TAG_*`) — what kind of thing it is.
- A set of **attributes** (`DW_AT_*`) — its properties (name, type, location…).
- Optional **children** — forming a tree.

```
   A DIE:
   ┌──────────────────────────────────────────────┐
   │ TAG:   DW_TAG_variable                       │
   │ ATTRS: DW_AT_name      = "count"             │
   │        DW_AT_type      = <ref to int DIE>    │
   │        DW_AT_location  = (rbx)               │
   │        DW_AT_decl_file = demo.c              │
   │        DW_AT_decl_line = 5                   │
   └──────────────────────────────────────────────┘
```

The whole `.debug_info` is a forest of these, one tree per **compilation unit**
(CU = one source file's worth of info).

---

## 6.2.2 The tree structure

```
 DW_TAG_compile_unit            "demo.c", language=C, producer="gcc", low_pc..high_pc
 │   DW_AT_name = "demo.c"
 │   DW_AT_comp_dir = "/home/u/proj"
 │   DW_AT_stmt_list → .debug_line offset (this CU's line table)
 │
 ├─ DW_TAG_base_type            "int", size=4, encoding=signed
 ├─ DW_TAG_base_type            "char", size=1, encoding=signed_char
 ├─ DW_TAG_pointer_type         → char       (i.e. "char *")
 │
 ├─ DW_TAG_variable             "g_init", type→int, location=(addr 0x4010), external
 │
 ├─ DW_TAG_subprogram           "main"           ← a function
 │   │   DW_AT_low_pc = 0x1149                    (start address)
 │   │   DW_AT_high_pc = 0x2f                     (length)
 │   │   DW_AT_frame_base = (DW_OP_call_frame_cfa)
 │   │   DW_AT_type → int                         (return type)
 │   │
 │   ├─ DW_TAG_formal_parameter "argc", type→int, location=...
 │   ├─ DW_TAG_variable         "count", type→int, location=(rbx)
 │   └─ DW_TAG_lexical_block    low_pc..high_pc   ← a { } scope
 │        └─ DW_TAG_variable    "i", type→int, location=...
 └─ ...
```

```
   Parent/child encodes SCOPE and CONTAINMENT:
     compile_unit ⊃ subprogram ⊃ lexical_block ⊃ variable
     structure_type ⊃ member ⊃ ...
   Sibling order encodes source order.
```

---

## 6.2.3 Common tags

```
 ┌────────────────────────────┬───────────────────────────────────────────────┐
 │ DW_TAG_compile_unit        │ a translation unit (root of a CU tree)        │
 │ DW_TAG_subprogram          │ a function (defined or declared)              │
 │ DW_TAG_formal_parameter    │ a function parameter                          │
 │ DW_TAG_variable            │ a variable (global, local, static)            │
 │ DW_TAG_lexical_block       │ a nested { } scope                            │
 │ DW_TAG_base_type           │ a built-in type (int, char, double)           │
 │ DW_TAG_pointer_type        │ T*                                            │
 │ DW_TAG_const_type / volatile_type │ cv-qualified type                      │
 │ DW_TAG_typedef             │ a typedef / using alias                       │
 │ DW_TAG_array_type          │ T[]  (with DW_TAG_subrange_type for bounds)   │
 │ DW_TAG_structure_type      │ struct / class                                │
 │ DW_TAG_class_type          │ C++ class                                     │
 │ DW_TAG_union_type          │ union                                         │
 │ DW_TAG_enumeration_type    │ enum (+ DW_TAG_enumerator children)           │
 │ DW_TAG_member              │ a struct/class field                          │
 │ DW_TAG_inheritance         │ a base class                                  │
 │ DW_TAG_namespace           │ C++ namespace                                 │
 │ DW_TAG_subroutine_type     │ a function type (for function pointers)       │
 │ DW_TAG_inlined_subroutine  │ a site where a function was inlined           │
 └────────────────────────────┴───────────────────────────────────────────────┘
```

---

## 6.2.4 Common attributes

```
 ┌─────────────────────┬───────────────────────────────────────────────────────┐
 │ DW_AT_name          │ the entity's name (offset into .debug_str)            │
 │ DW_AT_type          │ reference to another DIE (the type of this entity)    │
 │ DW_AT_low_pc        │ start address (functions, blocks)                     │
 │ DW_AT_high_pc       │ end address OR length (since DWARF4, usually length)  │
 │ DW_AT_location      │ a location expression / loclist (where data lives)    │
 │ DW_AT_decl_file/line/column │ source position                               │
 │ DW_AT_byte_size     │ size of a type                                        │
 │ DW_AT_data_member_location │ a field's offset within its struct             │
 │ DW_AT_encoding      │ base-type encoding (signed/unsigned/float/bool/...)   │
 │ DW_AT_external      │ flag: visible outside the CU                          │
 │ DW_AT_declaration   │ flag: a declaration, not a definition                 │
 │ DW_AT_frame_base    │ how to compute the frame base (often the CFA)         │
 │ DW_AT_specification │ links a definition DIE back to its declaration DIE    │
 │ DW_AT_abstract_origin │ links an inlined instance to the original function  │
 └─────────────────────┴───────────────────────────────────────────────────────┘
```

> **Types as a shared graph ▸** `DW_AT_type` is a *reference* (an offset to
> another DIE). So `int x; int y;` both point to the *same* `DW_TAG_base_type`
> "int" DIE. Pointer/const/array types chain: `const int *` =
> pointer_type → const_type → base_type(int). This sharing is a big part of why
> DWARF stays compact.

---

## 6.2.5 The abbreviation mechanism (why DWARF is compact)

Storing the tag name and each attribute name+form inline for every DIE would be
huge. Instead, DWARF factors out a **schema**: `.debug_abbrev` holds
*abbreviation declarations*, and each DIE in `.debug_info` just starts with an
abbreviation *code* (a ULEB128), followed by raw attribute *values*.

```
   .debug_abbrev (the schema/template):
   ┌───────────────────────────────────────────────────────────────────┐
   │ abbrev #1: TAG=compile_unit, has_children=yes                     │
   │   AT_producer  FORM_strp                                          │
   │   AT_language  FORM_data1                                         │
   │   AT_name      FORM_strp                                          │
   │   AT_low_pc    FORM_addr                                          │
   │   AT_high_pc   FORM_data8                                         │
   │   AT_stmt_list FORM_sec_offset                                    │
   │ abbrev #2: TAG=variable, has_children=no                          │
   │   AT_name FORM_strp ; AT_type FORM_ref4 ; AT_location FORM_exprloc│
   │ ...                                                               │
   └───────────────────────────────────────────────────────────────────┘

   .debug_info (the data, referencing abbrevs by code):
   ┌─────────────────────────────────────────────────────────────────┐
   │ [abbrev 1] <producer offset> <lang> <name offset> <lowpc> ...   │  ← a CU DIE
   │   [abbrev 2] <name off> <type ref> <loc expr>                   │  ← a var DIE
   │   [abbrev 2] <name off> <type ref> <loc expr>                   │  ← another
   │   0  (null DIE = end of children)                               │
   └─────────────────────────────────────────────────────────────────┘
```

```
   To parse a DIE:
     1. read its abbrev code (ULEB128).
     2. look up that abbrev in .debug_abbrev → get the ordered (attr, form) list.
     3. read each attribute's value using its FORM (which dictates byte layout).
     4. if the abbrev says has_children, recurse until a null DIE (code 0).
```

The **form** (`DW_FORM_*`) tells you how to decode each value:

```
   DW_FORM_strp      4/8-byte offset into .debug_str
   DW_FORM_string    inline NUL-terminated string
   DW_FORM_data1/2/4/8  raw N-byte constant
   DW_FORM_udata/sdata  (S)LEB128 constant
   DW_FORM_addr      a target address (4/8 bytes)
   DW_FORM_ref4      4-byte reference to another DIE (CU-relative offset)
   DW_FORM_exprloc   a counted DWARF expression (a location)
   DW_FORM_sec_offset offset into another .debug_* section
   DW_FORM_flag_present  the flag is true (zero bytes!)
   DW_FORM_addrx/strx (DWARF5) index into .debug_addr/.debug_str_offsets
```

This abbreviation + form scheme is the single biggest reason `.debug_info` is as
small as it is, and the reason you can't read it without `.debug_abbrev`.

---

## 6.2.6 Reading the tree

**Try it ▸**

```bash
cat > t.cpp <<'EOF'
namespace ns { struct Point { int x; double y; }; }
ns::Point g{1, 2.0};
int add(int a, int b){ int s = a + b; return s; }
int main(){ return add((int)g.x, 3); }
EOF
g++ -g -O0 -gdwarf-5 t.cpp -o t
llvm-dwarfdump --debug-info t | sed -n '1,60p'
```

Annotated excerpt:

```
 0x0000000b: DW_TAG_compile_unit
               DW_AT_producer ("clang/gcc ...")
               DW_AT_language (DW_LANG_C_plus_plus)
               DW_AT_name     ("t.cpp")
               DW_AT_low_pc / DW_AT_high_pc

 0x00000030:   DW_TAG_namespace
                 DW_AT_name ("ns")
 0x00000035:     DW_TAG_structure_type
                   DW_AT_name ("Point")
                   DW_AT_byte_size (0x10)            ← sizeof(Point)=16 (padding!)
 0x0000003e:       DW_TAG_member
                     DW_AT_name ("x")
                     DW_AT_type (→ int)
                     DW_AT_data_member_location (0)  ← offset 0
 0x0000004a:       DW_TAG_member
                     DW_AT_name ("y")
                     DW_AT_type (→ double)
                     DW_AT_data_member_location (8)  ← offset 8 (after 4+4 pad)

 0x........:   DW_TAG_subprogram
                 DW_AT_name ("add")
                 DW_AT_low_pc (0x....)
                 DW_AT_type (→ int)                  ← return type
 0x........:     DW_TAG_formal_parameter DW_AT_name("a") ...
 0x........:     DW_TAG_formal_parameter DW_AT_name("b") ...
 0x........:     DW_TAG_variable          DW_AT_name("s") ...
```

You can literally see `Point`'s layout (member offsets 0 and 8, total size 16
including the 4-byte padding between `int x` and `double y` — exactly the
alignment story from §1.2.5). **DWARF is how a debugger knows your struct
layout.**

---

## 6.2.7 Location expressions: the variable-finding stack machine

`DW_AT_location` doesn't just say "offset 8". It's a tiny **stack-machine
program** (DWARF expressions, `DW_OP_*`) that the debugger evaluates to compute
*where* a value is. This is what makes DWARF able to track optimized variables.

```
   DW_OP_addr 0x4010          → the value is at absolute address 0x4010 (a global)
   DW_OP_reg3                 → the value IS in register rbx (rax=0,rdx=1,rcx=2,rbx=3..)
   DW_OP_breg6 -8             → at [rbp - 8]  (a stack local; breg6 = rbp + offset)
   DW_OP_fbreg -20            → at frame_base + (-20)  (relative to DW_AT_frame_base)
   DW_OP_call_frame_cfa       → push the CFA (canonical frame address; §6.4)
```

```
   Evaluating DW_OP_fbreg -20 for variable `s`:
     1. compute frame_base (often = CFA from CFI)
     2. push frame_base ; add -20 → the address of `s`
     3. read sizeof(int)=4 bytes there
```

For optimized code, a single variable uses a **location list** (`.debug_loclists`):
different expressions for different PC ranges, because the variable migrates
between registers and memory as the optimizer pleases:

```
   variable `count`:
     [0x1149, 0x1158): in register rbx        (DW_OP_reg3)
     [0x1158, 0x1170): on the stack [rbp-8]   (DW_OP_breg6 -8)
     [0x1170, 0x117f): OPTIMIZED OUT          (no location)
```

> **"value optimized out" ▸** When GDB says `<optimized out>`, it's reporting
> that the variable's location list has no entry for the current PC — the
> optimizer didn't keep it live there. This is correct DWARF, not a bug; compile
> with `-Og` or `-O0` for more complete locations.

---

## Summary

- `.debug_info` is a tree of **DIEs**; each DIE has a **tag** (`DW_TAG_*`),
  **attributes** (`DW_AT_*`), and optional children encoding scope/containment.
- Functions (`subprogram`), variables, parameters, scopes (`lexical_block`), and
  every type are DIEs; types form a shared, referenced graph (compactness).
- The **abbreviation** mechanism (`.debug_abbrev`) factors out per-DIE schemas;
  each DIE in `.debug_info` is just an abbrev code + values decoded by their
  **form** — this is why you need both sections together.
- `DW_AT_data_member_location`/`DW_AT_byte_size` expose exact struct layout
  (padding included) — how debuggers know your types.
- `DW_AT_location` is a **DWARF expression** (a stack machine); optimized
  variables use per-PC **location lists**, which is the source of
  `<optimized out>`.

Next: [6.3 — The line-number program](03-line-number-program.md)
