# 7.3 — C++ Name Mangling, vtables & COMDAT

C++ has features the linker (a C-era tool that only knows flat names) was never
designed for: overloading, namespaces, templates, virtual dispatch, RTTI,
inline. The **Itanium C++ ABI** bridges this gap with **name mangling**, **COMDAT
groups**, and well-defined layouts for **vtables** and **type_info**. This is the
most C++-specific chapter in the guide.

---

## 7.3.1 Why mangling exists

The linker matches symbols by **name only**. But C++ allows many entities to
share a source name:

```cpp
   int  add(int, int);          // these three are DIFFERENT functions
   double add(double, double);   // but all named "add"
   namespace math { int add(int,int); }   // and this one too
```

If all emitted the symbol `add`, the linker would see "multiple definition" or
link the wrong one. **Name mangling** encodes the full signature (namespace,
class, parameter types, cv-qualifiers, templates) into a unique symbol name.

```
   source                            mangled symbol (Itanium ABI)
   ──────                            ────────────────────────────
   add(int, int)                     _Z3addii
   add(double, double)               _Z3adddd
   math::add(int, int)               _ZN4math3addEii
```

> **`extern "C"` ▸** Disables mangling: the symbol is the bare name (`add`),
> with no overloading allowed. This is how C++ exposes C-callable APIs and how
> `dlsym("name")` finds an entry point. It's also why you can't overload an
> `extern "C"` function.

---

## 7.3.2 The Itanium mangling grammar (decode by hand)

The (almost universal on Linux/macOS) Itanium scheme is systematic. The
essentials:

```
   _Z              prefix: "this is a mangled C++ name"
   N ... E         a nested name (qualified): N <parts> E
   <number><chars> a "source-name": length-prefixed identifier
                     3add = "add", 4math = "math", 3Foo = "Foo"
   Parameter types follow the name, concatenated:
     i=int  j=unsigned  l=long  c=char  d=double  f=float  b=bool  v=void
     P<T>=pointer to T   R<T>=lvalue ref   O<T>=rvalue ref
     K<T>=const T        V<T>=volatile T
     S_, S0_, S1_ ...    "substitutions" (back-references to repeated types)
```

```
   Decode _ZN4math3addEii:
     _Z              mangled name
       N ... E       nested:
         4math       "math"          (4 chars)
         3add        "add"           (3 chars)
       i i           (int, int)      parameters
     → math::add(int, int)

   Decode _ZNK3Foo3getEv:
     _Z N K          nested, and K = const (this is a const member fn)
       3Foo 3get     Foo::get
       v             (void)          → no params
     → Foo::get() const

   Decode _Z3maxIiET_S0_S0_ :
     _Z 3max         max
        I i E         template args <int>
        T_ ...        return/param refer to template params (T_ = first param)
     → max<int>(int, int)
```

The **substitution** mechanism (`S_`, `S0_`, …) compresses repeated components
(common with `std::` types whose mangled names are enormous), which is why real
STL symbols look like dense soup.

**Try it ▸** Mangle and demangle:

```bash
echo '_ZN4math3addEii' | c++filt              # math::add(int, int)
echo '_ZNSt6vectorIiSaIiEE9push_backEOi' | c++filt   # std::vector<int>::push_back(int&&)
nm app | grep _Z | head ; nm -C app | head     # raw vs demangled
c++filt -t '3Foo'                              # demangle a type
```

> **MSVC differs ▸** The Microsoft C++ ABI uses a *completely different* mangling
> scheme (e.g. `?add@@YAHHH@Z`). That's a core reason MSVC and GCC/Clang objects
> can't be linked together. On Linux/macOS the Itanium ABI is the lingua franca.

---

## 7.3.3 vtables and the virtual call mechanism

Virtual functions are implemented with a **vtable**: a per-class array of
function pointers. Each polymorphic object holds a hidden **vptr** to its class's
vtable.

```
   struct Base { virtual void f(); virtual void g(); virtual ~Base(); };
   struct Derived : Base { void f() override; void h(); };

   object layout:                 vtable for Derived (in .data.rel.ro):
   ┌──────────────┐               ┌──────────────────────────────┐
   │ vptr ────────┼──────────────▶│ offset-to-top (0)            │
   │ Base members │               │ &typeinfo for Derived        │
   │ Derived ...  │               ├──────────────────────────────┤
   └──────────────┘               │ &Derived::f   (overridden)   │  ← slot 0
                                  │ &Base::g      (inherited)    │  ← slot 1
                                  │ &Derived::~Derived (complete)│ ← slot 2
                                  │ &Derived::~Derived (deleting)│ ← slot 3
                                  └──────────────────────────────┘

   call obj->g():
     mov  rax, [obj]          ; load vptr
     call [rax + 8]           ; call vtable slot 1 (g)  ← indirect dispatch
```

```
   The vtable layout (Itanium ABI):
   ┌────────────────────┐
   │ offset-to-top      │  ← for adjusting `this` in multiple inheritance
   │ typeinfo pointer   │  ← RTTI: used by dynamic_cast / typeid (at vtable[-1])
   │ virtual fn ptr 0   │  ◀── the vptr actually points HERE (the address point)
   │ virtual fn ptr 1   │
   │ ...                │
   └────────────────────┘
```

- The **vptr** points at the *first function pointer* (the "address point"), with
  RTTI and offset-to-top at *negative* offsets.
- Mangled vtable symbol: `_ZTV7Derived` (`_ZTV` = "vtable for"). RTTI:
  `_ZTI7Derived` (`_ZTI` = "typeinfo for"), `_ZTS7Derived` (`_ZTS` = "typeinfo
  name string").

> **The key function rule ▸** To avoid emitting the vtable in *every* TU, the
> Itanium ABI emits a class's vtable in the **single TU that defines its first
> non-inline, non-pure virtual function** (the "key function"). If a class has no
> key function (all virtuals inline), the vtable is emitted as weak/COMDAT in
> every TU that needs it. **Pitfall:** declaring a virtual but never *defining*
> it → no key function gets a body → vtable never emitted → `undefined reference
> to vtable for X` (one of the most confusing C++ link errors; the fix is to
> define the first out-of-line virtual).

---

## 7.3.4 COMDAT groups: deduplicating inline & templates

Templates and inline functions are defined in headers and thus compiled into
*every* TU that includes them. Without help, the linker would see "multiple
definition." **COMDAT groups** (section groups, `SHT_GROUP`) solve this: each
such definition is placed in its own group, tagged with a **signature** (the
mangled name). The linker keeps **one** group per signature and discards the
rest.

```
   a.o:  group "_Z3maxIiET_S0_S0_"  { .text.<max<int>>  ... }
   b.o:  group "_Z3maxIiET_S0_S0_"  { .text.<max<int>>  ... }   ← same signature
   link: keep a.o's group, DROP b.o's.  No "multiple definition".
```

```
   SHT_GROUP section:
   ┌──────────────────────────────────────────────┐
   │ flags (GRP_COMDAT)                           │
   │ [section index, section index, ...]          │  ← the sections in this group
   │ signature = a symbol (the mangled name)      │  ← dedup key (sh_info → symtab)
   └──────────────────────────────────────────────┘
```

These definitions are emitted as **weak** symbols (§5.2.2), so even legacy paths
tolerate duplicates. This is the mechanical basis of the C++ ODR for
inline/template entities.

> **The silent ODR trap (revisited) ▸** The linker keeps **one arbitrary** copy
> and assumes all copies are identical. If two TUs compiled `max<int>` (or an
> inline function, or a class with inline members) *differently* — different
> macros, different struct layout, different `-O`/`-DNDEBUG`, ABI flags — the
> kept copy may not match what other TUs expected. **No diagnostic; UB.** This is
> why mixing object files built with incompatible flags/headers is dangerous, and
> why tools like `gold --detect-odr-violations` and LTO's whole-program view
> exist.

**Try it ▸**

```bash
cat > hdr.h <<'EOF'
template<class T> T mymax(T a, T b){ return a>b?a:b; }
inline int helper(){ return 7; }
EOF
printf '#include "hdr.h"\nint ua(){return mymax(1,2)+helper();}\n' > ua.cpp
printf '#include "hdr.h"\nint ub(){return mymax(3,4)+helper();}\n' > ub.cpp
g++ -c ua.cpp ub.cpp
readelf -gW ua.o                 # COMDAT section groups + their signatures
nm -C ua.o | grep -E 'mymax|helper'   # weak (W) symbols
g++ ua.o ub.o -shared -o libt.so && echo "linked, deduped, no error"
```

---

## 7.3.5 RTTI and `type_info` identity across modules

`dynamic_cast` and `typeid` rely on `std::type_info` objects (`_ZTIxxx`).
**Type identity is decided by the *address* of the `type_info` object**, not by
comparing strings (a fast-path optimization). This causes a subtle cross-module
bug:

```
   If the SAME type's type_info is emitted (weak/COMDAT) in two independently
   loaded shared objects and their symbols are NOT unified, the two type_info
   addresses differ → dynamic_cast across the boundary FAILS, typeid compares
   unequal, and an exception thrown in one module isn't caught by a handler in
   the other.
```

```
   fixes:
   - export the type's RTTI with default visibility from ONE shared library that
     others depend on (so the symbol is unified at load via interposition);
   - link with -Wl,--dynamic-list-cpp-typeinfo;
   - STB_GNU_UNIQUE bindings (glibc) for inline statics/type_info;
   - avoid RTLD_LOCAL for plugins that exchange polymorphic types.
```

This is a frequent real-world plugin/`dlopen` bug and a direct consequence of how
the C++ ABI maps language features onto ELF symbols.

---

## 7.3.6 Other C++ ABI artifacts you'll meet in symbol tables

```
   _Z...           a function/variable
   _ZN...E         nested (namespace/class) name
   _ZTV<class>     vtable
   _ZTI<class>     typeinfo
   _ZTS<class>     typeinfo name (string)
   _ZTT<class>     VTT (virtual table table, for virtual inheritance ctors)
   _ZThn..._       a "thunk" (this-pointer adjusting trampoline for multiple inh.)
   _ZGV<name>      guard variable for a function-local static (__cxa_guard_*)
   __cxa_*         runtime ABI helpers: __cxa_throw, __cxa_atexit,
                   __cxa_guard_acquire, __cxa_pure_virtual, ...
   _GLOBAL__sub_I_*  per-TU static-initialization function (calls ctors → .init_array)
```

Recognizing these makes `nm`/`objdump`/backtrace output legible. For example, a
crash in `__cxa_pure_virtual` means a pure virtual was called during
construction/destruction; a `_ZGVZ...` symbol is a thread-safe static guard.

---

## Summary

- **Name mangling** encodes namespaces, classes, signatures, cv-qualifiers, and
  template args into unique ELF symbols so the C-era linker can handle C++
  overloading; the **Itanium ABI** scheme (`_Z...`) is decodable by hand and the
  Linux/macOS standard (`extern "C"` opts out; MSVC differs entirely).
- **vtables** (`_ZTV`) are per-class function-pointer arrays; the **vptr** points
  at the address point with RTTI/offset-to-top at negative offsets. The **key
  function** rule decides which TU emits the vtable — and explains the "undefined
  reference to vtable" error.
- **COMDAT groups** (`SHT_GROUP`, weak symbols) deduplicate inline/template
  definitions across TUs, mechanically implementing the ODR — and silently
  keeping one arbitrary copy when definitions disagree (dangerous UB).
- **RTTI/type_info identity is by address**, causing cross-module
  `dynamic_cast`/exception failures unless symbols are unified.
- Recognizing `_ZTV/_ZTI/_ZThn/_ZGV/__cxa_*/_GLOBAL__sub_I_*` makes symbol
  dumps and backtraces readable.

Next: [7.4 — Security mitigations](04-security-mitigations.md)
