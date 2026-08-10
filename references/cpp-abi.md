# C++ ABI reference: mangling, vtables, and modelling them in Ghidra

Lookup detail for C++ targets. `SKILL.md` carries the triggers — *establish the
inheritance model before relying on slot positions*, *check `ClassUtils` before
hand-rolling a vftable struct* — and this file is what you consult once one fires.

## MSVC mangled names

Shape: `?<member>@<Class>@@<access><callconv><return><args>Z`, classes innermost-first,
so `?Foo@Inner@Outer@@...` is `Outer::Inner::Foo`.

The character immediately after `@@` encodes access **and virtualness** — the single most
useful byte in the name. *(Standard MSVC `undname` encoding, not verified against this
Ghidra install; the `Q` and `U` rows were confirmed live against a real binary, the rest are
from the published scheme.)*

| Char | Meaning |
|---|---|
| `A` / `B` | private, non-virtual |
| `E` / `F` | **private virtual** |
| `I` / `J` | protected, non-virtual |
| `M` / `N` | **protected virtual** |
| `Q` / `R` | public, non-virtual |
| `U` / `V` | **public virtual** |
| `S` | public **static** |
| `Y` | free function (no class) |

A function occupying a vtable slot must carry a virtual specifier (`E`/`F`/`M`/`N`/`U`/`V`).
A `Q` or `S` in a slot means your table is not a vtable, or your slot attribution is wrong.

### Special names (`??` prefix)

| Symbol | Meaning |
|---|---|
| `??0Class@@` | constructor |
| `??1Class@@` | destructor |
| `??2` / `??3` | `operator new` / `operator delete` |
| `??_7Class@@6B@` | **vftable** — its address IS the class's vtable, slot 0 |
| `??_7Derived@@6BBase@@@` | **secondary vftable** → multiple inheritance |
| `??_8Class@@7B@` | **vbtable** (virtual base table) → **virtual inheritance** |
| `??_9` | **vcall thunk** → MI dispatch adjustment |
| `??_B` | function-local static initialisation guard (**not** inheritance) |
| `??_C` | string literal |
| `??_E` / `??_G` | vector / **scalar deleting destructor** |
| `??_R0`…`??_R4` | RTTI descriptors (absent when built `/GR-`) |

**Inheritance-model test.** Only unqualified `??_7Class@@6B@` forms present, with **no
`??_8` and no `??_9`**, is positive evidence of a plain single-inheritance chain — the
precondition for any slot-index reasoning.

### The destructor trap, learned the hard way

**A virtual destructor's vtable slot holds a compiler-generated *deleting destructor*
thunk, not `~Class` itself.** So `~Class` never appearing as a raw vtable target is
*expected ABI behaviour, not a scan gap*. In this project a mangled-vs-vtable cross-check
scored 188/199, and every one of the 11 misses was this. Budget for it before concluding a
sweep is incomplete.

**Which thunk it is must be measured, not assumed — this reference previously asserted
`??_G` and the binary disagreed.** Decompile one and read the flag parameter:

| Branch present | Thunk | Mangling |
|---|---|---|
| `flags & 1` only, no loop | **scalar** deleting destructor | `??_G` |
| `flags & 2` array loop striding by the object size, count at `*(this-4)`, **plus** `flags & 1` | **vector** deleting destructor | `??_E` |

Measured on this 1999 MSVC binary: **all nine** anchored classes' slot-38 targets are the
**vector** form, with per-class strides (0x38, 0x2a8, 0x578, …). Naming them "scalar" would
have asserted what the decompiled body disproves. The array branch is the discriminator.

### Comparing names — the string-space trap

Raw mangled names (`?Sleep@CGobject@@QAEXXZ`) and Ghidra's demangled short names (`Sleep`)
are **different string spaces that never intersect**. A check comparing one against the
other reports 0/N forever and reads as a failing sweep rather than a broken test. Convert
first. Likewise `Function.getName()` is unqualified while `getName(True)` includes the
namespace.

Ghidra 12.1 added Microsoft demangler **output options** controlling user-defined-type tags
(`struct` etc. in argument positions) and anonymous-namespace presentation
(`_anon_ABCD01234` vs the generic form). Both affect whether demangled and non-demangled
symbols land in the same namespace, so they affect name-based joins.

## Itanium ABI (GCC/Clang — Linux, most consoles' modern toolchains)

| Symbol | Meaning |
|---|---|
| `_Z...` | mangled function |
| `_ZTV<class>` | vtable |
| `_ZTI` / `_ZTS` | typeinfo / typeinfo name |
| `_ZTT` | VTT — **virtual inheritance present** |
| `_ZThn...` / `_ZTv...` | non-virtual / virtual adjustor thunks → **multiple inheritance** |
| `_ZGV` | guard variable |

Itanium vtables carry an offset-to-top and RTTI pointer *before* slot 0, so the symbol
address is not the first function pointer — unlike MSVC. Account for the header before
computing slot indices.

## Other toolchain families — do not assume MSVC or Itanium

The two schemes above cover Windows and most Unix/modern-console targets, but not the
console era this skill also targets:

- **Metrowerks CodeWarrior** — the dominant compiler for **GameCube, Wii**, and a large
  share of **PS2** titles, plus older Mac. Its C++ mangling is its own scheme, broadly
  `name__<qualifiers><argtypes>` with `F` introducing the argument list and `__` separating
  the member name from its class — e.g. `foo__3BarFi` for `Bar::foo(int)`. Ghidra's MSVC and
  Itanium demanglers will **not** decode it, so mangled names may pass through unrecognised
  and read as opaque garbage rather than as symbols. Its vtable and RTTI layout also differ.
  If mangled-looking names in a GameCube/Wii/PS2 binary decode under neither shipped
  demangler, suspect CodeWarrior before concluding the names are stripped.
- **Watcom** — DOS-era games. Its own mangling, and `__watcall` register-based calling
  conventions that differ from every convention listed under x86 elsewhere.
- **Borland / Turbo C++** — DOS/early-Windows era, another distinct scheme.

Practical consequence: **identify the compiler before interpreting symbols or calling
conventions.** Rich data ("Rich header") in a PE, linker version fields, CRT banners, and the
shape of function prologues all help. Getting this wrong makes every symbol look absent and
every argument look misplaced.

## Modelling a C++ class in Ghidra

### Use `ClassUtils` — it is a convention, not a helper bag

`ghidra.program.model.gclass.ClassUtils` / `ClassID`:

```python
ClassUtils.isVTable(dataType)
ClassUtils.getVftDefaultEntry(dtm) / getVftEntrySize(dtm)     # vftable slot type/size
ClassUtils.getVbtDefaultEntry(dtm) / getVbtEntrySize(dtm)     # virtual BASE table
ClassUtils.getClassPath(classId)
ClassUtils.getClassInternalsPath(composite)   # category path for "class internals"
ClassUtils.getSelfBaseType(composite)
ClassUtils.getBaseClassDataTypePath(composite)
ClassUtils.getReplacementPointers(dtm, structure)
ClassUtils.getReplacementType(structure)
ClassUtils.hasClassAttribute(structure)
ClassUtils.createVxTableDescriptionOffsetTag(ptrOffsetInClass)
ClassUtils.validateVtableDescriptionOffsetTag(description)
```

Following these keeps recovered classes interoperable with Ghidra's PDB and RTTI
machinery. Hand-rolled structs sit parallel to it and cannot be consumed by it.

Related: `SymbolTable.createClass(parent, name, sourceType)` creates a `GhidraClass`, which
Ghidra auto-associates with a struct of the same name.

### Getting virtual method names into the decompiler

Ghidra's official recipe (from `improvingDisassemblyAndDecompilation.pdf`):

1. Create a **`FunctionDefinitionDataType`** per virtual method, calling convention
   `__thiscall`.
2. Create a **`<Class>_vftable` structure** whose fields are those function definitions
   **in slot order**, each field **named** after its method.
3. Make the **class struct's first field a pointer** to that vftable struct.
4. Then `Auto Fill in Structure` on a known `this` fills remaining fields from access
   patterns.

Shortcut for finding the tables in the first place: `Search → For Address Tables`, then
apply the definitions to the pointers and `Data → Create Structure`.

### Devirtualization — three explicit mechanisms

Ghidra does **not** devirtualize automatically; it cannot prove a vtable pointer is
constant after assignment (issues #650, #516). Escalating:

1. **Name the slots** — the recipe above. Yields `this->vtable->Method(...)` rendering.
2. **Mark the vtable data `CONSTANT`** (`MutabilitySettingsDefinition`: `NORMAL`,
   `CONSTANT`, `VOLATILE`, `WRITABLE`; per-data via Settings or per-block via Memory Map).
   *The decompiler shows the contents of constant memory rather than a pointer to it* —
   which is what lets it read through the table. `.rdata` genuinely is read-only, so this
   is factually correct rather than a hack.
3. **Override the call site** — add a primary reference of type
   `RefType.CALL_OVERRIDE_UNCONDITIONAL` at the indirect call (relatives:
   `JUMP_OVERRIDE_UNCONDITIONAL`, `CALLOTHER_OVERRIDE_CALL/JUMP`). Converts it to a direct
   call. Ghidra 12.1 did something equivalent for Objective-C — `WhatsNew.md` states that
   `_objc_msgSend` calls "have been overridden to reference the actual target method (if
   discoverable)" — though it does not name the mechanism, so treat the specific `RefType`
   attribution as inference, not documented fact.

**Only mechanism 3 changes what the call *is*.** Mechanisms 1 and 2 make the table and its
slots legible; the call site stays indirect and no call-graph edge appears. If you need
readable decompilation, 1 and 2 suffice. If you need call-graph edges — call trees,
reachability, "who can invoke this" — you need 3, per site.

**Mechanism 3 writes your inference into the call graph.** Tag it `SourceType.AI`, record it
as an assertion, and never let a harvest read it back as fact.

## Slot-index correspondence — free names, under one precondition

Under single inheritance, a derived class's vtable is its base's slot sequence with
overrides substituted in place and new virtuals appended. Therefore: **if a base table's
slot *i* holds a named exported virtual, and a derived table's slot *i* holds `FUN_xxxx`,
that function is the override of that method** — name it `Derived::Method`. This is ABI
mechanics, not inference.

Preconditions, all of which must be checked, not assumed:

- Single inheritance (the `??_8`/`??_9`/qualified-`??_7` test above).
- Correct slot alignment between the tables — join on **target addresses**, not demangled
  short names, since an override shares its base's name and makes different tables look
  identical.
- The base table must be complete; a table truncated by an undefined-function pointer
  shifts nothing but hides slots beyond the truncation.
