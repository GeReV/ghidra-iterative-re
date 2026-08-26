# C++ ABI reference: mangling, vtables, and modelling them in Ghidra

Lookup detail for C++ targets. `SKILL.md` carries the triggers — *establish the
inheritance model before relying on slot positions*, *check `ClassUtils` before
hand-rolling a vftable struct* — and this file is what you consult once one fires.

## MSVC mangled names

Shape for a **non-static member function**:

```
?<member>@<Class>@@ <access> <cv> <callconv> <return> <args> Z
```

classes innermost-first, so `?Foo@Inner@Outer@@...` is `Outer::Inner::Foo`.

**The `<cv>` character is easy to miss and its absence corrupts everything after it.**
`?Sleep@CGobject@@QAEXXZ` parses as `Q` (public non-virtual) `A` (cv-qualifier: none) `E`
(**`__thiscall`**) `X` `X` `Z` — *not* `Q` + `A`(=`__cdecl`). A parser written against a
grammar with no cv slot reads the cv byte as the calling convention and reports
`__cdecl` for **every `__thiscall` method in the binary**. Static members and free
functions have no `this` and so no cv byte.

Calling conventions (Ghidra's `MDCallingConvention`): `A`/`B` `__cdecl`, `C`/`D` `__pascal`,
**`E`/`F` `__thiscall`**, `G`/`H` `__stdcall`, `I`/`J` `__fastcall`.

The access character immediately after `@@` encodes access, virtualness **and staticness** —
the single most useful byte in the name. *(Full table from Ghidra's own
`MicrosoftDmang` source, `MDTypeInfoParser.java`. Earlier revisions of this file listed only
8 of the 13 rows, which silently mis-classified private/protected statics and had no entry
at all for adjustor thunks.)*

| Char | Meaning |
|---|---|
| `A` / `B` | private, non-virtual |
| `C` / `D` | private **static** |
| `E` / `F` | **private virtual** |
| `G` / `H` | private **adjustor thunk** |
| `I` / `J` | protected, non-virtual |
| `K` / `L` | protected **static** |
| `M` / `N` | **protected virtual** |
| `O` / `P` | protected **adjustor thunk** |
| `Q` / `R` | public, non-virtual |
| `S` / `T` | public **static** |
| `U` / `V` | **public virtual** |
| `W` / `X` | public **adjustor thunk** |
| `Y` / `Z` | free function (no class) |

**Data symbols have their own code, and it decides OWNERSHIP:** `0`/`1`/`2` =
private/protected/public **static class member**, **`3` = true global**, `4` = static local,
`5` = guard, `6` = vftable (or RTTI4), `7` = vbtable, `8` = RTTI. So `?pRendEng@@3PAV…` is a
global, while `?Table@CFoo@@2PAV…` is `CFoo`'s static member — and calling the second a
global is an ownership error of the same family as mis-attributing a field. Measured on one
binary: **50 class statics against 17 true globals**, i.e. the naive reading is wrong for
the majority.

**Vtable slots.** Under **single inheritance**, a function occupying a vtable slot carries a
virtual specifier (`E`/`F`/`M`/`N`/`U`/`V`), and a `Q` or `S` in a slot means your table is
not a vtable or your slot attribution is wrong. **Under multiple inheritance that rule is
false**: a secondary vtable's slot holds an **adjustor thunk** (`G`/`H`, `O`/`P`, `W`/`X`) —
a stub that fixes `this` by a fixed offset and jumps to the real override, e.g.
`sub ecx,8; jmp CDerived::Foo`, mangled `?Foo@CDerived@@W7AEXXZ`. So the rule holds only
given the inheritance-model test below, and a slot-index naming scheme run on an MI binary
would name the *stub* while the real override stayed anonymous.

**Adjustor-thunk absence is therefore a second, independent inheritance-model test.** A
census of access characters costs one pass: zero `G/H/O/P/W/X` corroborates single
inheritance from a different direction than "no `??_8`, no `??_9`", because under single
inheritance `this` never needs adjusting and there is nothing for a thunk to do. Measured on
one binary: 0 of 1094 mangled symbols — agreeing with the `??_8`/`??_9` zero.

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
| `??_D` | vbase destructor |
| `??_E` / `??_G` | vector / scalar deleting destructor — **MSVC puts `??_E` in the vtable**, Clang puts `??_G` (see the destructor trap below) |
| `??_F` | default-constructor closure |
| `??_H` / `??_I` | **vector constructor / destructor iterator** — the ABI's own witness that a member is an embedded **array**, and a far cheaper one than deriving element size from loop strides |
| `??_O` | copy-constructor closure |
| `??_S` | local vftable |
| `??_U` / `??_V` | `operator new[]` / `operator delete[]` — note an **array** allocation's size includes the count prefix, so it is not `n * sizeof(T)`; relevant wherever allocation size is used as a size bound |
| `??_R0`…`??_R4` | RTTI descriptors (absent when built `/GR-`) |

**`.?AV` is not only an RTTI signal.** MSVC emits `_TypeDescriptor` records for types named
in `catch` and `throw` **independently of `/GR`**. So a measured zero says *neither
polymorphic RTTI nor typed C++ EH* — a stronger finding than "no RTTI" — and `.?AV` present
in a `/GR-` build is exception data, whose surrounding structures are `FuncInfo`, not
`??_R0`. That `FuncInfo` is itself worth harvesting: its unwind map **names a destructor per
stack object with that object's frame offset**, and its try/catch map names caught types.

**Inheritance-model test.** Only unqualified `??_7Class@@6B@` forms present, with **no
`??_8` and no `??_9`**, is positive evidence of a plain single-inheritance chain — the
precondition for any slot-index reasoning.

### The destructor trap, learned the hard way

**A virtual destructor's vtable slot holds a compiler-generated *deleting destructor*
thunk, not `~Class` itself.** So `~Class` never appearing as a raw vtable target is
*expected ABI behaviour, not a scan gap*. In this project a mangled-vs-vtable cross-check
scored 188/199, and every one of the 11 misses was this. Budget for it before concluding a
sweep is incomplete.

**For MSVC the slot holds the VECTOR deleting destructor (`??_E`), not the scalar one.**
This reference previously asserted `??_G`; that is wrong for MSVC and the error is worth
understanding, because it is the single most likely thing to mislead you here:

> *"MSVC's virtual tables always contain a pointer to the vector deleting destructor for
> classes with virtual destructors … because clang always puts a pointer to a scalar
> deleting destructor to the vtable."* — [LLVM PR #170337](https://github.com/llvm/llvm-project/pull/170337)

So **the two toolchains genuinely differ**, and "the slot holds `??_G`" is true of
Clang/Itanium-style output and false of MSVC. Check which compiler produced your binary
before trusting either statement.

The flags parameter, which is how you identify the form from a decompiled body
([Chen](https://devblogs.microsoft.com/oldnewthing/20040203-00/?p=40763),
[Shilon](https://ofekshilon.com/2014/06/09/on-vector-deleting-destructors-and-some-newdelete-internals/)):

| Test | Meaning |
|---|---|
| `flags & 1` | also free the memory |
| `flags & 2` | destroying an **array**; otherwise a single object |

The element count is *"hidden in front of the vector"* — stored immediately before the
array, which is why the array branch reads it from `*(this - 4)` on 32-bit.

**One function serves both roles**: the vector deleting destructor behaves as a scalar one
when `flags & 2 == 0`, so the presence of the `& 2` branch identifies the *generated form*,
not what a particular call site does. (Shilon notes he has *"never seen a vector deleting
destructor called with flags different from 3"* in practice.)

Confirmed on one 1999 MSVC binary: all nine anchored classes' slot-38 targets carry the
`& 2` array branch with per-class strides (0x38, 0x2a8, 0x578, …) — consistent with the
rule above. Note the binary exported **no `??_E` or `??_G` symbol at all** (only `??_7` and
`??_B`), so it could not adjudicate its own naming; the identification came from the body
shape plus the documented MSVC rule, not from a symbol.

**The converse trap, and it bites when you go looking for the SCALAR destructor.** There are
**two** destructor bodies per polymorphic class, and the obvious way to recognise either one
recognises *both*: the scalar `~X` and the vector deleting destructor **both install the
class vptr into `[this+0]`, and both call the base-class destructor.** A recovery rule built
from "installs this class's vptr, then calls an ancestor's destructor" is therefore
structurally unable to separate them, and it will confidently hand you the deleting
destructor under the name `~X`.

Measured: a program-wide sweep for scalar destructors proposed `CRobotPart::~CRobotPart` for
a body that was in fact the slot-38 **deleting** destructor of a *different* (unnamed) class —
wrong class and wrong kind — and it was caught only because that name had been correctly
applied to another address one round earlier. The discriminator costs one predicate and was
already known from the deleting-destructor work above:

> a body carrying the **`flags & 1` test** *and* a **call to `operator delete`** is the
> DELETING destructor; the scalar one has neither.

Two general points fall out of that incident. **When a rule targets a shape, enumerate the
other populations that share the shape and encode the exclusion**, rather than waiting for a
collision to reveal it — here the exclusion already existed in a sibling round's applier and
simply had not been carried across. And **put the discriminator in the probe that first
proposes the answer, not only in the applier that would refuse it**: a pricing probe's output
is read, quoted and scoped against long before any applier sees it.

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

- **The two tables are in an ANCESTOR–DESCENDANT relation, and the slot index lies within
  the ancestor's table.** This is the precondition that gets dropped, because "slot *i* in
  both tables" is trivially computable and the relationship is not. New virtuals are
  *appended*, so every branch of a hierarchy starts numbering its own additions at the same
  index — two classes in **sibling** branches are therefore **guaranteed** to hold unrelated
  methods at the same slot, not merely likely to. Formally: slot *i* is comparable between
  classes A and B only if their nearest common ancestor's table has **more than *i***
  entries. Above that boundary the index carries no information at all.

  **Measured, on a 1999 MSVC/x86 game.** A round was scoped, priced and approved on the
  rule without this test, keyed on the hierarchy **root** and slot index — a key that looked
  conservative and was not, because **223 of 224 tables rooted at one class**, so "per
  hierarchy" was the global rule wearing a hat. It produced **111 candidate names; exactly
  1 survived** the ancestor-length test. The shape of the collision: the shared ancestor's
  table held 152 slots, one branch grew to 171 and another to 215, so every index past 152
  named two different methods depending on the branch — and two of the round's headline
  groups (14 candidates each) were a single 47-vs-53-slot collision counted twice. *Check
  the distribution of whatever you key on before trusting the key.*

  **Calibrate this on ground truth before trusting either direction, and note that the
  population you need contains no candidates at all**: take every PAIR of tables that
  carry a mangled export name at the same slot — both ends ground truth, no inference
  anywhere — and split by whether the slot is comparable between them. On the binary
  above: **1,957,540 comparable pairs agree and 0 disagree; 10,526 cross-branch pairs
  disagree and 0 agree.** The regimes separate *perfectly*, which is the useful shape of
  the result in both directions — off-branch the index is noise rather than a weak signal,
  and on-branch the propagation is confirmed nearly two million times over, which is a far
  better warrant for applying it than any candidate-level argument. Make that split an
  assertion that raises if either side empties or either regime stops separating.

  **And before pricing this route at all, check whether the project has already APPLIED
  it.** In the incident above the repo had shipped an applier using the correct
  ancestor-form rule 56 program versions earlier, carrying its own calibration in its
  header; the round that priced the loose rule had not read it. An applier is a rule plus
  its calibration plus the population it deliberately abstained from — a pricing probe
  written from scratch starts from none of those and can regress against all three.

- **A second witness that is blind to the hierarchy — on MSVC x86 the cheapest is the
  `RET imm16`.** `__thiscall` makes the callee pop its arguments, so the terminal return
  instruction's immediate **is** the argument byte count, straight from the instruction
  stream. In the incident above it refuted **60 of 108** readable candidates on its own,
  knowing nothing about tables or names, and passed the single survivor. Note the asymmetry
  before using it as confirmation: most virtuals take 0 or 1 argument, so *agreement* is
  what two unrelated functions do and only *disagreement* carries information.

- Single inheritance (the `??_8`/`??_9`/qualified-`??_7` test above).
- Correct slot alignment between the tables — join on **target addresses**, not demangled
  short names, since an override shares its base's name and makes different tables look
  identical.
- The base table must be complete; a table truncated by an undefined-function pointer
  shifts nothing but hides slots beyond the truncation.

  **A truncated table does not merely hide slots — it forges a depth, and depth is what
  ancestry arguments are built on.** Measured here: a 47-slot table was recorded as 42
  because slot 42's target had real instructions but no defined `Function`. At 42 it looked
  *shallower* than the 47-slot classes it actually equals, and a depth-ranking test
  therefore promoted it to a nearer ancestor of four classes — inverting the ancestry. At
  its true 47 it is not a candidate at all. Before any depth comparison, verify each
  table's extent independently of the sweep that produced it: walk past the recorded end
  and check whether code pointers continue, and where the incoming references actually fall.

## Recovering *which* class derives from which

The `??_8`/`??_9` test above establishes the inheritance **model** (single vs virtual vs
multiple). It does not tell you the **graph** — who derives from whom. Those are different
questions and the second one has a trap.

### Slot similarity ranks; it does not order

The tempting approach is to compare tables: a derived table shares its base's slot prefix,
so rank candidate bases by how many leading slot targets they share. That works as a
*tie-break* and fails as a *primary source*, because **similarity is symmetric and
inheritance is not**. Nothing in "these two tables share 43 slots" says which is the
parent. Depth is then used to supply the missing direction — and that is where it goes
wrong.

> **Never require the parent to be strictly shallower than the child.** A derived class
> that adds **no new virtuals** has *exactly* its base's slot count. The rule feels safe —
> it prevents sibling cycles — but it makes a same-depth parent **structurally ineligible**,
> so the ranking silently returns the *grandparent* and every check downstream agrees with
> it, because the grandparent really is an ancestor.
>
> Measured on one binary: three 47-slot classes were assigned to their 38-slot grandparent
> because their real parent was also 47 slots, differing only at its own destructor slot
> and one other. Nothing failed. The chain was self-consistent, passed a cycle check, and
> passed a 100%-agreement correspondence gate — the wrong answer is compatible with all of
> them, because an ancestor's slots correspond too.

### Constructors carry the direction

Use the binary's own construction order instead. A constructor builds its bases first, so
with the base constructor inlined it **stores each base's vftable into offset 0 of the
object in turn**, before storing its own:

> **{ classes whose `??0`/`??1` stores table V } == V's class + its descendants**

That relation is *directional*, comes from code plus exported symbols rather than from
shape comparison, and is cheap: find `MOV dword ptr [reg], <imm32 == V>` with no
displacement, take the containing function, and read its mangled name.

Combine them in the right order: **ancestry decides who is eligible, similarity picks the
nearest among them.** Then a same-depth parent is admitted naturally and no depth rule is
needed anywhere.

Caveats, both load-bearing:

- **It is a LOWER BOUND.** Inlining depth varies, so a missing pair proves nothing while a
  present pair is strong. Never read absence as evidence of non-ancestry.
- **A base's vftable store and a first-member-subobject store look identical** — both write
  a vftable to offset 0. Guard with the ABI invariant that a base is never *deeper* than
  its derived class, and treat a cycle in the relation as proof the detection is matching
  something else.

### Typing `this` from a call site — an UPPER BOUND on depth

Tempting shortcut, once some registers are typed: at a direct `CALL` whose `ECX` holds a
provably typed object, the callee's `this` is that type. It is the `__thiscall` contract
and it reaches non-exported bodies that have no signature of their own. **Measured on one
binary, it is wrong a third of the time**, and the failures are two recognisable shapes:

- **The caller holds a DERIVED pointer and calls an INHERITED method.** Inferred
  `CVehicle*`, declared `CBasicUnit*`; inferred `CStructure*`, declared `CBasicGobject*`.
  So an inferred receiver type is an **upper bound on depth** — the mirror of the
  lower-bound rule above, and the same direction problem as the destructor stride.
- **The callee is not `__thiscall` at all**, so `ECX` was never `this`: a private
  *static* member (`?F@C@@CA...`) inherited a caller's register type. Where the callee is
  exported you can filter this by convention — and where it is *not* exported, which is
  exactly the population the shortcut exists to reach, you cannot.

Two consequences. First, calibrate it before use: many inferred callees are themselves
exported and carry a mangled declaring class, so the inference can be checked against
ground truth for free (36 agreed, 18 disagreed — refuted). Second, the depth error is
survivable where the *offset* decides ownership: on a single-inheritance chain each
offset lies in exactly one class's own range, so attributing an access by offset rather
than by receiver type is correct even when the receiver is named too deep. Attribute
by offset; use the receiver only to pick the chain.

### How to tell you got it right

A corrected graph should make an *independent* measurement better, not just different.
Here, re-deriving raised the slot-correspondence sample count 89 → 107 while agreement
stayed at 100%: the new samples came from the newly-corrected edges, so wrong parentage
would have shown up as disagreement. Consistency proves little on its own; a rival
measurement improving is worth much more.

---

## Dispatch recovery — virtual calls are one case of several

*Moved here from `SKILL.md` in the 2026-08-23 restructure; the mechanics it points
at were already in this file.*

Do not force an OOP reading onto a program that isn't OOP. Era matters: pre-C++ and
early-C++ game code dispatches through plain function-pointer tables far more often than
through vtables, and the recovery mechanics are the same while the *semantics* are not.

**Establish the inheritance model before relying on slot positions.** A derived class's
vtable shares a prefix with its base only under single inheritance, so any naming or
ownership scheme built on slot index depends on it. For MSVC this is three cheap string
searches, not an assumption:

| Search | Meaning if found |
|---|---|
| `??_8` | **vbtable** — virtual inheritance is present; slot prefixes are not reliable |
| `??_9` | **vcall thunk** — multiple-inheritance dispatch adjustment |
| `??_7Derived@@6BBase@@@` (qualified form) | a **secondary** vftable, i.e. multiple inheritance |

Only the unqualified `??_7Class@@6B@` form appearing, with no `??_8` and no `??_9`,
is positive evidence of a plain single-inheritance chain. (Measured on one 1999 MSVC game
binary: 14 unqualified vftables, zero `??_8`, zero `??_9` — single inheritance confirmed
for every class carrying an exported vftable symbol, which is a floor for the rest.)
Itanium-ABI equivalents: `_ZTV` vtables, `_ZTT` VTT for virtual inheritance, `_ZThn`/`_ZTv`
thunks.

Shapes to expect, all of which look alike in `.rodata`:

- **C++ vtables** — one per class, slot order = declaration order, derived tables share a
  prefix with their base. Only these carry inheritance meaning.
- **State-machine / message-handler tables** — indexed by state or message enum. The
  index is a *grammar*, not a class.
- **Opcode / bytecode interpreter dispatch** — a table indexed by opcode. Recovering it
  recovers the scripting language; each handler names one instruction.
- **Callback registration structs** — `{name, fnptr, arity}` triples. Often self-naming.
- **Jump tables from switch statements** — Ghidra may fail to bound the switch variable;
  `FindUnrecoveredSwitchesScript.java` locates these and `SwitchOverride.java` fixes them.

A run-of-code-pointers heuristic cannot tell these apart. **Discriminate by how the table
is used**, not by its shape: a C++ vtable's address gets stored into offset 0 of an object
by a constructor; a dispatch table is indexed by a variable at a call site. Verify against
any authoritative naming (vftable symbols, RTTI) before believing an interpretation.

### If it *is* C++

Three things to know here; the mechanics are in **`references/cpp-abi.md`**.

- **`ClassUtils` exists** (`ghidra.program.model.gclass`) — Ghidra has a *convention* for
  modelling C++ classes with vtables, including virtual base tables. Follow it and your
  results interoperate with Ghidra's PDB/RTTI machinery; hand-roll and they sit parallel to
  it, unusable by it. **Check it before building any vftable struct by hand.**
- **Ghidra does not devirtualize automatically** — it cannot prove a vtable pointer is
  constant after assignment (issues #650, #516). But four explicit mechanisms do work,
  escalating: name the slots via a vftable struct, mark the table `CONSTANT` so the
  decompiler reads *through* it, run **`AddVfunctionCallRefScript`** (shipped in 12.1 —
  see the measured caveats below, they are severe), or override the call site with
  `RefType.CALL_OVERRIDE_UNCONDITIONAL`. The last two **write your inference into the call
  graph** — tag them `SourceType.AI` and never let a harvest read them back as fact.
- **`AddVfunctionCallRefScript`'s real preconditions, and why batching it is unsound.**
  An earlier revision of this document said its precondition was "a single corresponding
  applied vftable structure, so exactly the work the struct-apply rounds already did."
  **That is wrong, measured against a 12.1.2 install and a project that had done exactly
  that work.** Read the source before planning a round on it:
  - `isVftableStructure()` requires **every component to be a `Pointer` to a
    `FunctionDefinition`**. A struct built the idiomatic way — from
    `ClassUtils.getVftDefaultEntry(dtm)`, which returns a plain `PointerDataType` — fails
    this on every component. Measured: **0 of 15** applied vftable structs passed. Naming
    the *fields* after the methods does not help either; `getOrdinalOfFunction()` matches
    the decompiler token against the **pointed-to type's** name.
  - It is **cursor-driven** (`currentLocation instanceof DecompilerLocation`) — one call
    site per invocation. There is no batch mode to "run and count".
  - It writes `SourceType.**ANALYSIS**`, not `AI`. On a project that filters harvests by
    tier, that launders an inference into the trusted tier.
  - **The soundness is supplied by the human at the cursor, and automation removes it.**
    Its inference is *"type T is applied at exactly one address A, therefore slot k
    resolves to `*(A + 4k)`"* — which assumes **static type == dynamic type**. A
    `Base *` at runtime points at a *derived* class's table. The script never checks for
    descendants because an analyst looking at one site knows whether the receiver can be
    derived. Measured on one hierarchy: sound for 6 of 14 classes, and the 8 unsound ones
    were the interesting ones, carrying **748 of 1066** slots (one base had 16 descendants).
    A batch version would emit confidently wrong call-graph edges while every report
    counted them as successes.

  Generalise it: **before batching any shipped interactive aid, ask what the human at the
  cursor was contributing.** These scripts are written for an analyst who supplies context
  the code does not check, and the missing check is usually invisible in the output.
- **"There is no provable devirtualization here" is a claim about your MECHANISM until you
  have tried the dataflow APIs.** A round in this project concluded devirtualization was
  impossible and gave the reason as "real devirtualization needs reaching-definitions
  dataflow to track `this` across basic blocks". Ghidra ships that:
  - `DecompilerUtils.getBackwardSlice` / `getBackwardSliceToPCodeOps` /
    `getForwardSlice(ToPCodeOps)` / `getDataTypeTraceForward|Backward`. `HighFunction` p-code
    is **SSA with `MULTIEQUAL` phis**, so a slice crosses basic blocks by construction;
    `Varnode.getDef()` / `getDescendants()` / `getLoneDescend()` are the raw def-use edges.
  - `SymbolicPropogator` (+ `ConstantPropagationContextEvaluator`) runs over raw
    instructions with no decompiler and no signature dependency, and its `Value` exposes
    **`isRegisterRelativeValue()` / `getRelativeRegister()`** — it models `this + K`
    natively, making it structurally independent of any decompiler-based witness.
    **IT IS NOT READ-ONLY, AND IT LAUNDERS INTO THE `ANALYSIS` TIER.** Verified on a 12.1.2
    install: `SymbolicPropogator.java:2725-2730` creates every recovered reference with
    `instruction.addMnemonicReference(target, refType, SourceType.ANALYSIS)` /
    `addOperandReference(...)`, and the only veto is a `ContextEvaluator` — `evaluateReference`
    at :2735-2740 opens with `if (evaluator == null) { return target; }`, a non-null return, so
    **running it with no evaluator writes references at the analyzer tier**. That is the same
    laundering shape this document refuses in `AddVfunctionCallRefScript`, in a tool it otherwise
    recommends. To use it as a pure witness, pass a `ContextEvaluator` whose `evaluateConstant`
    and `evaluateReference` return `null`; `ContextEvaluator` is a plain interface, so
    `@JImplements` works from PyGhidra. Other gotchas: it needs `recordStartEndState=true`
    (docstring-confirmed) — *the claim that `saveContext=true` is ALSO required for
    `getRegisterValue` is **UNVERIFIED**, and reading :186-219 with :344-359 suggests it may not
    be; measure it before repeating it* — and its recording default changed in **11.4.1**
    (`docs/ChangeHistory.md:678`, GP-5804), **not** 12.0 as an earlier revision of this document
    said.
  - Interprocedural templates ship as scripts: `ShowConstantUse.java` ("walk backward
    through function calls to find any constants that find their way directly into the
    variable") and `WindowsResourceReference.java`.
  - Heavier, also shipped: the `TaintAnalysis` module, the `SymbolicSummaryZ3` extension,
    `ExportPCodeForCTADL.java`, and a **LiSA** abstract-interpretation extension with a
    runnable launch script.

  A negative result is only as strong as the strongest mechanism you actually ran. Name the
  mechanism in the finding, or the next round inherits a conclusion it cannot audit.
- **A VPTR-STORE CHAIN NAMES ANCESTORS, NOT PARENTS — the optimizer deletes the links you
  need most.** The parentage rule at the end of this section ("get parentage from constructors,
  not from table similarity")
  is right about *direction* and silently optimistic about *immediacy*. MSVC writes
  `*this = &vftable` once per class in each constructor and each destructor — and then
  **deletes any such store nothing can observe before the next one overwrites it.** An
  intermediate class contributes a link if and only if something between its store and the
  next store can see the vptr. Measured on one 1999 MSVC/x86 binary, in both directions
  within one round:
  - **Survives:** a factory storing base table `B`, then `CALL <ctor>` with `this` in ECX,
    then its own table. The call could read the vptr, so `B` stays.
  - **Elided:** a factory whose intermediate constructor was INLINED, so the sequence is
    grandparent-store, plain field initialisers, own-table store. The intermediate's write is
    dead and gone — and the derived class's records name the **grandparent** as its parent,
    self-consistently and forever. Proof the write existed: the same class also has a
    standalone out-of-line constructor, never called from that factory, performing the
    identical field initialisations *and* the store.

  The destructor side is identical: three classes' deleting-dtor thunks all called the same
  out-of-line destructor, which reset the vptr to a table **two levels up**, because the
  intermediate classes' destructors were trivial and their stores dead. An agent reading only
  destructors and an agent reading only constructors produced contradictory parents for the
  same six classes, each with a clean derivation.

  So: **a vptr chain always yields a true ANCESTOR and never proves an IMMEDIATE base, and a
  missing link is not evidence of a missing class.** Consequences worth acting on:
  - Label the column honestly — *nearest observed ancestor* — or a later round will "correct"
    correct data. Where slot-prefix similarity says a nearer table exists (equal or near-equal
    slot counts, a large shared-slot margin), that is the signal to look for an elided store,
    not a contradiction to resolve by picking a winner.
  - **Recover the elided link from the class's own out-of-line constructor**, which usually
    exists even when every factory inlines it. Matching *field-initialiser sets* between the
    inlined and out-of-line forms is what ties them together.
  - **Two agents disagreeing is a finding about the MECHANISM, not a tie to break.** Both
    derivations here were sound; the ranking question ("which is closer?") was the wrong
    question, and asking it would have discarded one correct answer.
- **A LEAFNESS RULE THAT READS VPTR STORES ALONE CANNOT SEE THE ORDINARY SUBCLASS.** "Does any
  body store this class's table and then a different one?" is the natural way to ask whether a
  class has a subclass, and it is only half the question. MSVC emits a derived constructor as
  `call base_ctor; mov [this], own_vftable`, so the base's vptr store lives in the **base's**
  body and never appears in the derived one — a store-only rule therefore sees only a subclass
  whose base constructor was **inlined**, and returns `LEAF` for every class whose children were
  compiled the normal way. Measured on one binary: it returned `LEAF` for the class with **four**
  subclasses while giving the right answer for the three classes that really are leaves, which
  is exactly why nothing looked wrong. **Classify a body by its vtable stores AND its calls to
  the class's constructor**, and take the demonstration from ground truth rather than a
  constructed poison: a class the binary itself declares subclasses for is a must-fire arm that
  costs nothing and cannot go inert.
- **"IS THERE A FUNCTION AT THIS ADDRESS" IS A CLAIM ABOUT THE DATABASE, AND IT DOES NOT BELONG
  IN A STRUCTURAL TEST.** This document already says a cross-reference count is a fact about the
  database rather than the binary. The same mistake hides one rung down, inside predicates that
  feel like observations: a "is this immediate a vtable?" test written as *every slot is a
  defined function entry point* refused **three genuine tables** on one binary, whose slot
  targets were real code in bytes Ghidra had never disassembled. Test what the binary
  guarantees — *the slots point into executable memory, at distinct addresses* — and **print
  each slot's kind** (`function` / `code` / `undisassembled` / `not-executable`), so the missing
  definitions stay visible as findings instead of being swallowed by the relaxation that admits
  them. Those slots are usually a real population worth a round of their own.
- **A DIFFERENT HIERARCHY CAN USE DIFFERENT SLOT NUMBERS, AND A SLOT-KEYED SWEEP REPORTS A CLEAN
  ZERO ON IT.** A slot index is a per-hierarchy convention, not a program-wide one. Measured: a
  binary's main object tree put `Save`/`Load` at slots **45/46**, a second, unrelated family put
  them at **0/1**, and every sweep keyed to the first numbering passed silently over the second
  — which was then written up across three rounds as "these classes are non-polymorphic, which
  is why every vtable-driven route is blind to them". The classes had vtables the whole time.
  **Before concluding a population has no vtable, ask which slot you were looking in.**
- **CLOSE A BLINDNESS GUARD ON EVERY ROUTE THE CLAIM RESTS ON, NOT JUST THE SEARCHABLE ONE.**
  Byte-searching for a vtable's address finds vptr stores hidden in undisassembled code; it can
  **never** find a hidden `E8 rel32` call, because the encoding is position-dependent. A guard
  covering only the searchable half reads as complete and is not. The other half is cheap and
  exact: enumerate the undisassembled ranges inside executable blocks (`Listing.getUndefinedRanges`
  over `Memory.getExecuteSet()` — it returns an `AddressSet`, so iterate `getAddressRanges()`)
  and decode every relative call in them. Measured on one binary: **5,317 ranges / 88,842 bytes**,
  and both halves paid — the store search found two real undisassembled destructors, and the call
  decode returned a *measured* zero instead of an assumed one.
- **A PREMISE RECORDED IN A NOTES FILE IS AN UNTESTED CLAIM, AND EVERY ROUND BUILT ON IT INHERITS
  IT WITHOUT RE-READING IT.** The self-harvest rules here are about evidence; this is the same
  failure on the *scoping* axis, and it is cheaper to fall into because nothing is ever written
  down twice. Measured: one round's parenthetical — "almost certainly non-polymorphic, which is
  precisely why every vtable-driven route here is blind to them" — scoped the next three rounds,
  one of which declined a rule that would have decided two open sizes *because* of it. It was
  refuted by a decompilation printed in one of those very rounds. The tell was not subtle and it
  was not missed for lack of evidence; it was missed because the premise had stopped being a
  question. **When three rounds in a row inherit the same negative premise, re-derive it from the
  program before the fourth** — and prefer premises stated as a probe that can be re-run over
  premises stated as a sentence.

- **Three byte-identical functions that were NOT folded are a cheap measurement that ICF is
  off.** "Identical-code folding hides overrides" is a real hazard and it is also frequently
  asserted without being checked. If a build emits per-class deleting-destructor thunks with
  identical bodies at distinct addresses, `/OPT:ICF` was not in effect, and every "X does not
  override Y" claim stops needing the ICF caveat. One decompile of two sibling thunks settles
  it.
- **"Zero cross-references" is a claim about the DATABASE, not the binary.** A reference from
  bytes the disassembler never made into a function is invisible to the reference model, so
  `getReferencesTo` returns an honest, empty, wrong answer — and the natural reading of that
  zero ("compiled but never instantiated", "dead code") is a strong conclusion built on a
  tool limitation. Measured: a vtable reported as having zero references of any kind had
  exactly one, inside an out-of-line constructor Ghidra had not recognised, findable only by
  byte-searching the image for the address. The blind population is precisely the undefined
  code that function-discovery rounds exist to convert — i.e. it shrinks as the project
  progresses, which is why the claim looks safer than it is. Byte-search for the address
  before recording any reference-count zero; it costs seconds.
- **Slot-index correspondence gives free names**, under single inheritance: if a base
  table's slot *i* is a named exported virtual and a derived table's slot *i* is `FUN_xxxx`,
  that function *is* the override. ABI mechanics, not inference — but check the
  preconditions in the reference, and expect virtual destructors to sit in slots as
  compiler-generated deleting-destructor thunks rather than as `~Class`.
- **Get parentage from constructors, not from table similarity.** Similarity is symmetric
  and inheritance is not, so ranking candidate bases by shared slots needs *something* to
  supply direction — and using depth for it is a trap: a derived class that adds no new
  virtuals has exactly its base's slot count, so a "parent must be strictly shallower" rule
  makes the real parent ineligible and silently returns the **grandparent**, self-consistently.
  A constructor stores each base's vftable into offset 0 in turn, which *is* directional.
  Mechanics and the measured failure: `references/cpp-abi.md`, "Recovering *which* class
  derives from which".

---

## Recover the INTERFACE before the classes, and read the roots, not the leaves

When a family of classes shares a vtable layout, the unit worth reading is **the interface, not the
class**. Measured on one 1999 MSVC/x86 binary, over 45 related GUI classes:

| | |
|---|---|
| distinct method bodies across the 45 tables | 188 |
| supplied by the two root tables | **33** |
| subclass overrides | 155 |
| slots a median class overrides | **4** |

The two roots turned out to **share 15 of their 20 base slots outright** — identical target
addresses — so one derived from the other and 33 bodies explained all 188. A round priced at "220
unnamed functions" was really priced at 33.

The order that follows from this:

1. **Diff the tables against each other first.** Equal target address at equal slot = inherited;
   different = override. That partitions the whole family before a single body is opened, and it
   costs one join over the vtable dump.
2. **Read the root's slots.** Each one you understand is understood for every class in the family —
   and if your citation system has a `slot k of table T` witness, each is *citable* for all of them
   at once.
3. **Read the overrides last.** They are the class-defining behaviour, and they only make sense once
   you know what the slot they replace is *for*.

### Identity is an OUTPUT of this, not an input

The tempting move is the opposite one: name the classes first, so the methods have somewhere to
live. It inverts the dependency. **A vtable is unattributed precisely because no name-supplying
route reaches it** — no RTTI, no mangled export naming the type, no registration string — so the
only evidence about it is what its methods DO. Naming the class first means inferring it from
behaviour you have not read yet.

Check before you start whether a name route really is exhausted: on that family, **no exported
mangled signature named a single one of the 51 unattributed tables**, and confirming that took one
join and settled the question.

### Validate an interface reading against a SUBCLASS before applying any of it

A claim about slot *k* is a claim about every class in the family, so it is worth far more than one
body's worth of care — and it is cheap to test, because the overrides are independent evidence.

Worked: slot 13 was read as "a child element told me it was chosen" from an **empty** base
implementation plus its two call sites. Two override bodies confirmed it outright — each compared
its argument against the child pointers that class had stored at construction and dispatched
accordingly. In the same pass, one of those overrides fetched a string by calling **slot 11** on a
child, which independently confirmed a `GetText` reading taken minutes earlier from a four-byte
body.

Three shapes worth recognising while reading such a family:

- **A slot that recurses into a child list through a fixed vtable displacement tells you its own
  index.** `(**(code **)(*child + 0x40))()` inside slot 16 is slot 16 calling itself on children —
  a free self-consistency check on your slot numbering.
- **An empty body (`ret`) in the base is a hook, not a stub.** Name it from its callers and its
  overrides, or leave it: a hook with no observed caller has no recoverable meaning.
- **A flag word tested by the resource/branch logic is the key to the verbs.** Once you know which
  bit selects the "disabled" artwork, the method that sets that bit is `Disable` — witnessed rather
  than guessed.

### The same trap, one round later, with the rule already written down

**This section was written on 2026-08-26 from the incident above. On 2026-08-27 the next
project round walked into the same trap anyway, and the failure is worth more than the rule.**

A round built a fresh evidence-pack generator that mapped vtable slot -> method name for a
45-class GUI family, and merged TWO roots into one map. The family is four branches: a 20-slot
base, and siblings of 28, 24 and 21 slots each extending it independently. The merged map
propagated the 28-slot branch's `Open`/`Close`/`SelectByCommandId` onto panels and buttons that
have no such methods. **7 of 61 mechanically-derived names would have been confidently wrong.**

Nothing above is new — the rule states the sibling case as *guaranteed*, not likely. What is new
is **why the rule did not fire**:

- **The round never asked the question, because the slot map came from an ARTIFACT JOIN.** The
  earlier incident keyed on the hierarchy root and got caught pricing candidate names. Here the
  map was assembled by a producer as a lookup table, and a lookup table has no step at which
  anyone asks *"is this index within the common ancestor?"* **Encode the precondition in the
  data structure, not in the analyst's attention**: key the map by `(branch_root, slot)`, never by
  `slot` alone. The bug is a `dict` that should have been two.
- **The check is one artifact column.** Comparing the ancestors' vtable LENGTHS settles it, and
  most projects already store the length. It costs one read and it was not taken.
- **Where a class's own branch has no decided name for a slot, SKIP — do not borrow.** Borrowing
  across the boundary is free, silent, and produces exactly the confident-wrong-name outcome the
  trust model exists to prevent. List the skipped addresses so the gap is a backlog item.

**What caught it was `RET imm16` — the second witness this section already recommended.** A
delegated reader counted pushed arguments in the disassembly: the 24-slot branch's slot 21 takes a
`char *` and copies it into a field, while a screen calls its slot 21 with nothing pushed, and one
virtual index cannot carry both signatures. **A join over a slot table structurally cannot see a
signature** — it sees occupancy only — so no amount of artifact cross-referencing would have found
this. That is the general rule (*a producer only finds what its witness kinds can see*) landing on
the producer itself.

**The transferable lesson is not about vtables.** A rule written in a reference file is not a rule
applied; it fires only where something in the workflow ASKS it. When a lesson's remedy is "check
X before Y", the durable fix is a structure or a check that cannot proceed without X — here, a
selftest pinning the branch boundary in both directions, so a merged map fails loudly instead of
producing plausible names.

