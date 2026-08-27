# External oracles, and target ABI vs host ABI

*Reference for the `ghidra-iterative-re` skill. Only `SKILL.md` is loaded when the
skill is invoked; this file is read on demand when its trigger fires. New lessons of
this kind belong here, not in `SKILL.md`.*

**Read this when:** you want a check the program cannot influence — a
recompile-and-diff, a second class recovery, a compiled header — or you are emitting a
layout that a 64-bit host will silently relay.

**In this file:**

- The external oracle — the only check that is not the program grading itself
- Target ABI vs host ABI — the width that breaks three things

## The external oracle — the only check that is not the program grading itself

Everything in the loop is internal: the program adjudicating claims about itself. Independent
witness kinds help, invariants help, poisons help — but they are all computed from the same
database by the same agent, and a systematically wrong assumption survives all of them. Four
routes step outside that, ordered by how directly they falsify a layout claim.

**1. Recompile and diff — the decision procedure.** Compile a candidate function with a
period-correct toolchain and diff its object code against the original bytes. This is not an
opinion, and a struct is validated *as a side effect*: a wrong field offset moves a
displacement and the diff shows it. Adopt the annotation convention early even if a full
match is never the goal — address-tagged source (`// FUNCTION: <module> 0x…`) turns an
unbuildable reimplementation into a growing set of falsifiable claims.

The mature toolchain for this on **32-bit x86 MSVC** is `isledecomp/reccmp`
(<https://github.com/isledecomp/reccmp>), built for exactly that profile, with siblings that
check the things this skill otherwise verifies only against itself: **`vtable`** (virtual
table correctness), **`datacmp`** (globals), **`stackcmp`** (stack layout). `isledecomp/isle`
is a fully decompiled MSVC 4.20 game and an honest calibration of what "matching" achieves
(functionally identical, mostly instruction-matching, *not* byte-for-byte).
**`encounter/objdiff`** is the x86-capable interactive differ. **Do not reach for the famous
N64-lineage tools** — `m2c`, `decomp-permuter` and `asm-differ` are MIPS/PPC/ARM/SH and do
not apply to x86; recommending them here is a category error.

**2. A second, independent class recovery.** Ghidra ships one —
`RecoverClassesFromRTTIScript` plus the `classrecovery` package (hierarchy, ctor/dtor
identification, vftable ordering, class structures filled from decompiler p-code store
information, with `ApplyClassFunctionSignatureUpdatesScript` to propagate a signature edit
through the vftable definitions). It **hard-gates on RTTI**, so on a `/GR-` binary it does
nothing — *and that is the citable justification for hand-rolling, which you should state
rather than leave implicit.* Where RTTI is absent, **CMU SEI Pharos `OOAnalyzer`** targets
"32-bit x86 executables compiled by Microsoft Visual C++" with an explicit `--ignore-rtti`,
emitting members and offsets, method↔class assignment, inheritance, ctors/dtors and
vftables; import it through **CERT Kaiju**. Read its published accuracy before believing it:
constructors ≈0.88 F1 and vftables ≈0.97, but **destructors 0.41 F1 (precision 0.32)** and
**no published accuracy for member offsets at all**. Import as *candidates*, never as
decided — and know that identical-code folding merges classes, so an ICF-heavy binary
degrades it further.

**3. A similarity witness that states its own error rate.** See the BSim VT correlator under
"Separating library code from game code" (`references/game-recon.md`): corpus-free, and it
publishes a false-positive model. Nothing else here does.

**4. `ProgramDiff` against a checkpoint version** — the change-set measurement under
"Mutation safety — the rest of the bracket" (`references/applying-changes.md`). Not an
external *source*, but it is the program's own record of what you did rather than your
memory of it.

**AND IF YOU HAVE ONE, WIRE IT TO SOMETHING THAT RUNS IT — AN UNWIRED ORACLE ROTS SILENTLY AND
NOTHING ELSE CAN NOTICE.** Measured, and it is the worst-placed instance of this document's "a
harness is blind to what it does not run" rule: a project that emits its recovered layouts as C
with `offsetof` assertions and compiles them — the recommendation under "Emit a C header"
in `references/applying-changes.md` — had that header
**failing to compile for 90 commits** and three program versions, 161 errors. It was found only
because an unrelated side project ran the generator for a baseline. The cause is worth carrying
on its own:

- **A TARGET POINTER IS 4 BYTES AND THE HOST'S IS NOT.** A round correctly retyped a field to
  `char *`, and the evidence artifact recorded it correctly — `ctype "char *"`, **`width 4`**,
  right for a 32-bit target. The generator emitted the *spelling* verbatim and ignored the
  *width*, so on any x86-64 host the member became 8 bytes and 8-ALIGNED, the containing struct
  went 268 → 280, and all five classes embedding it shifted. **When an artifact records both a
  spelling and a width, the width is the fact.** Emit fixed-width types (`uint32_t` for a 32-bit
  target pointer, real type in a comment) and the header stays host-independent.
- **HOST-INDEPENDENCE IS WHAT MAKES THE CHECK RUNNABLE, NOT A STYLE PREFERENCE.** The obvious
  alternative — `-m32` — failed twice on that machine: at link (`Scrt1.o` absent) and even under
  `-fsyntax-only` (`bits/libc-header-start.h`), for want of multilib. **A check that needs a
  toolchain you do not have is a check that will not be run**, which is how it came to rot.
- **The sibling generator had already made the right decision and documented it.** Two emitters,
  one convention, one follower — the copy-instead-of-move divergence "A rule built for one
  population must be offered to its SIBLING populations" (`references/harvesting-traps.md`)
  warns about, between two files nobody thought of as sharing a rule.

**Why this is a standing instruction:** a methodology built around never corroborating your
own guesses should want, more than anything, one check it cannot influence. If none of these
is reachable, say so explicitly in the round's record — "no external oracle available" is a
finding about the project's confidence ceiling, not an absence worth passing over in silence.

## Target ABI vs host ABI — the width that breaks three things

Nearly every target this document is about is 32-bit, and nearly every machine you will build on
is 64-bit. Under LP64 a pointer is **8 bytes and 8-ALIGNED** where the target's is 4, and `long`
is 8 where the target's is 4. Write `char *` or `long` into a recovered struct and the compiler
silently relays every field after it and changes `sizeof`.

The same mistake surfaces in three places, in increasing order of cost and decreasing order of
noise — which is exactly the wrong order for finding it.

**1. In the HEADER it is loud, and that is the cheap case.** Measured: one field retyped from
`uint` to `char *` — correctly; the evidence artifact recorded `ctype "char *"` with **`width
4`** — was emitted with its *spelling* rather than its *width*. `sizeof` of the containing record
went **268 → 280**, the five classes embedding it all shifted, and the header produced **161
errors for 90 commits** because nothing was wired to run the compile. **When an artifact records
both a spelling and a width, the width is the fact.** Emit `uint32_t` for a 32-bit target pointer
and name the real type in a comment. The header then compiles anywhere, which is the difference
between a check that runs and a check that rots.

**2. In a RECOMPILE-AND-DIFF it is quiet, and it invalidates the comparison.** Every displacement
in the emitted object code is computed from the host's layout, so a struct the host resized
produces instruction-level differences that are artifacts of your build rather than of your
reimplementation — and you will chase them as if they were findings. Build 32-bit for this work
(`-m32` plus the multilib, or a cross-toolchain), and establish that you *can* before planning
around it: on one machine `-m32` failed at link for `Scrt1.o` and even under `-fsyntax-only` for
`bits/libc-header-start.h`.

**3. In a SOURCE-LEVEL REIMPLEMENTATION it is quiet and load-bearing**, because the properties
such projects are graded by are precisely the byte-layout-sensitive ones: a save format, a replay,
a network packet, a **lockstep sync checksum**. A reimplementation built 64-bit reproduces the
*behaviour* of every struct containing a pointer and none of its *layout* — so it desyncs against
the original binary and cannot load its own saves, for a reason that never appears anywhere in the
logic and survives every unit test you would think to write. **If the destination is functional
equivalence graded by those oracles, the pointer width is part of the specification.** Model
target pointers as explicit 32-bit handles or offsets, or commit to building 32-bit — and write
down which, because the decision is invisible in the source once made.

**A WIDER SCALAR SMUGGLES IN AN ALIGNMENT THE SAME WAY A POINTER SMUGGLES IN A SIZE.** The
pointer case above is the famous half; this is the one that will bite you when you emit *gaps*.
Measured, on the very first run of a generator written with the pointer lesson already in hand:
the obvious width→type map `{1: uint8_t, 2: uint16_t, 4: uint32_t, 8: uint64_t}` put a `uint64_t`
on an 8-byte **gap** at offset `0x1dc`. That offset is 4-aligned and not 8-aligned, so the
compiler padded to `0x1e0` and relayed every later field in the class. **An 8-byte gap is not a
64-bit integer — it is eight bytes nothing observes**, and the recovered evidence claimed a width,
never an alignment. Emit gaps and any width you did not independently establish as a scalar as
`uint8_t[N]`, which aligns to 1 and can therefore sit at whatever offset the evidence records.
The general form: **every type name is a claim, and a layout emitter must only make the claims its
evidence actually supports.**

Two things worth taking from how that surfaced:

- **The external evaluator grades your PIPELINE, not only your data.** That round had honestly
  priced itself as "assurance, not a findings engine" — 3753 of 3796 fields were naturally
  placeable and no class had non-abutting regions, so nothing was expected to fire. The first
  compile rejected the *emitter*. Build the oracle even when you expect the data to be clean.
- **Ask the compiler WHERE it diverged; do not read the declarations and reason.** The failing
  assertions named offsets from `0x1f4` onward and the declarations there looked perfectly
  consecutive and 4-aligned. A probe that walks the members in order and reports the FIRST
  mismatch named the real culprit 24 bytes earlier, immediately.

**The general rule, and it is not only about pointers:** a recovered layout is a statement about
the TARGET's ABI. Anything whose size or alignment differs between target and host — pointers,
`long`, `size_t`, `ptrdiff_t`, enum width, `double` alignment, bitfield packing order, struct tail
padding — must be pinned explicitly rather than inherited from whatever compiler happens to build
your source. Ghidra will tell you what the target thinks: the `data_organization` block in the
compiler spec (`x86win.cspec` for 32-bit MSVC) records `pointer_size`, `long_long_size`,
`double_size`, the size→alignment map and `bitfield_packing use_MS_convention`. That is the
authority; your build host is not.

Endianness and fixed-point are the same family of problem on non-x86 targets and are covered in
`references/platforms-eras.md`.

## The format's own integrity checks ARE the oracle — find them first

An external oracle is the only check that is not the program grading itself, and the usual ones are
expensive: a shipped file to round-trip, a companion tool, a second implementation. Before reaching
for any of those, **look for the checks the format performs on ITSELF.** A serialised format written
by a careful team usually validates as it reads, and those validations encode the invariants the
authors considered load-bearing — which is exactly what a reimplementation must preserve.

**Measured**, on a 1999 game's save format. The loader guards every object record three ways:

- a leading sequence number that must equal a running counter,
- a trailing sequence number that must equal it again,
- and, after populating the object, **`ftell() != recorded_end_offset -> bail`**.

That third check asserts the single most useful invariant available: **the reader consumed exactly
the bytes the writer produced.** And it runs **once per OBJECT**, so a failure localises to one
class rather than to the file — a far sharper instrument than "does the save load".

- **Grep the bail/assert/panic strings early.** Five strings fenced this entire format
  (`"Savegame File Invalid!"`, `"Corrupted loadgame file"` at three sites, `"Undefined object type
  during load game"`, `"Expected object non-existant during loadgame"`, `"Cannot save a non-handlized
  object!"`), and each one names an invariant for free. In a release build with asserts compiled
  out, these survive because they are error paths, not asserts.
- **Prefer a per-record check to a whole-file one** when grading a reimplementation. Pass/fail on a
  file tells you something is wrong; a per-object bound tells you which class.
- **A length or end-offset field is the check.** Any format that records where a record ends is
  telling you it intends to verify that, or to skip — either way it hands you a boundary you can
  assert against.

## An in-binary check can corroborate a TOOLING conclusion by an unrelated route

When your instruments say a thing about the binary, the program's own assertions may say the same
thing independently — and that pairing is worth much more than either alone, because the two routes
share no failure modes.

**Measured.** A probe comparison had flagged 19 classes whose serializer and deserializer appeared
to touch different fields. Reading the bodies concluded they were all artefacts of the probe. The
loader's own `ftell() != end_offset` check then said the same thing from inside the program: a
genuine field-written-never-read would make the game unable to load its own save, so the disagreement
could not have been real. Neither route was cheap to doubt afterwards.

Look for this deliberately: after concluding "our tool was wrong, not the binary", ask **what the
binary would have to do at runtime if we were wrong** — and whether it checks for it.

## A back-patched length field tells you the reader's shape

`write placeholder -> write payload -> seek back -> write the real position -> seek forward` is a
distinctive idiom, and it is not just bookkeeping. A format that records where each record ends was
designed for a reader that can **skip a record it does not understand.**

Seeing it, predict the rest before reading it: a dispatch loop keyed on a type tag, a registry
mapping tag to constructor, and at least one pass in which the payload is **not** read at all.
Measured here: all three were present, and the loader turned out to be **two-pass** — construct
every object from its header while seeking past every payload, then rewind and populate. That second
structure follows from objects referencing each other by handle, and it is worth checking for
whenever a format stores cross-references.
