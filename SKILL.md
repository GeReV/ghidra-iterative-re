---
name: ghidra-iterative-re
description: Use when reverse-engineering a game (or any binary) in Ghidra via MCP or PyGhidra - establishes the apply-cascade-verify loop, the SourceType trust model that stops an agent corroborating its own guesses, invariant bracketing against Ghidra's silent collateral damage, and the game-specific evidence sources worth sweeping before any analysis.
---

# Iterative reverse engineering in Ghidra

Ghidra is not a disassembly viewer you read from. It is an inference engine with a
persistent, versioned database. Knowledge you apply changes what the decompiler shows
you next, which is where the next round of knowledge comes from. Refusing to write to it
throws away the tool's main mechanism and is the more expensive choice.

But the loop has a failure mode batch analysis does not: **you can corroborate your own
guesses.** Apply an inference, re-read, and the confirming read looks independent when it
is not. Part 1 makes the loop safe. Part 2 is what to point it at in a game. Part 3 is
the tool surface.

**`references/api.md` holds the verified API surface** — class and method names checked
against a real install, plus the offline doc paths to discover more. Read it before
writing scripts; it exists because recalling Ghidra's API from memory produces plausible
names that do not exist, and because several of the most useful classes are absent from
the public javadoc.

---

# Part 1 — The loop (any binary)

## Rounds

```
checkpoint → apply (highest-certainty tier only) → cascade (scoped)
           → verify invariants → harvest → adjudicate → next round
```

Stop after two consecutive rounds produce nothing new — not when you run out of ideas.

**Apply in certainty order, never convenience order.** Ground truth from the binary
first, then mechanical derivations from it, then inferences. A round that applies a guess
before an available certainty has corrupted every round after it.

Ghidra's own advanced course says of the **Decompiler Parameter ID** analyzer: *"If you
run this analyzer too early or before fixing problems, you can end up propagating bad
information all over the program."* The cascade propagates errors as well as facts.

## The trust model — the load-bearing part

Ghidra tracks provenance natively via `SourceType`. Priority, highest first:

| SourceType | Means | Usable as evidence? |
|---|---|---|
| `USER_DEFINED` | A human decided this | Yes, but record that a human is the source |
| `IMPORTED` | From the binary: exports, demangler, debug info | **Yes — ground truth** |
| `ANALYSIS` | Ghidra's analyzers inferred it | Yes, with the analyzer named |
| `AI` | *"content produced through AI assistance"* — **yours** | **NEVER** |
| `DEFAULT` | `FUN_`/`DAT_` placeholders | No — absence of information |

**Tag every mutation you make `SourceType.AI`.** The anti-circularity rule then stops
being a discipline and becomes a query filter:

> **Harvesters must exclude `SourceType.AI` symbols from evidence.**

A round's own applications then structurally cannot become evidence for the claim that
motivated them. Verify the filter fires: tag one known symbol `AI`, confirm it drops out
of the harvest.

### Mutate through scripts, not MCP tools

**Audited in GhidrAssistMCP source (`jtang613/GhidrAssistMCP@master`): every mutating
tool hardcodes `SourceType.USER_DEFINED` — 18 occurrences across `RenameSymbolTool`,
`CreateFunctionTool`, `CreateDataVarTool`, `SetFunctionPrototypeTool`, `StructTool`,
`VariablesTool`. `SourceType.AI` appears nowhere.**

An MCP mutation therefore launders your inference into the *highest* provenance tier,
indistinguishable from a human decision and permanently uncleanable from evidence. **Do
all mutation from scripts.** MCP read-only queries are fine. Audit any other MCP server
the same way before trusting it.

## Mutation safety

**Every mutation is bracketed by an invariant diff, measured before and after, that
`raise`s on change. Never prints — raises.**

A recorded incident: `ApplyFunctionDataTypesCmd` with a broad address set against a large
type archive **destroyed two unrelated functions** — disassembly and function definition
gone, bytes reverted to undefined — with **no error, no exception, no log line.**
`applyTo()` returned cleanly. Caught only by diffing function count (2640 → 2638).

- Pick a cheap whole-program invariant (function count usually) and diff it across every
  mutating operation.
- Count the same way every time and write down which way. Ghidra has more than one
  function count and they differ (`getFunctions(True)` excludes externals;
  `getFunctionCount()` includes them).
- **A clean return and no exception is not evidence nothing was damaged.**
- **Scope address sets as narrowly as the operation allows.** Broad set + large archive
  is the dangerous combination; the incident did not reproduce narrowly.
- Gate mutating scripts behind an explicit `apply` argument so a bare run is a dry run.
- **Non-returning functions**: if Ghidra doesn't know a function never returns, it
  disassembles the data after every call to it, which "wreaks havoc." Check Error
  bookmarks; `FixupNoReturnFunctionsScript.java` lists candidates and repairs damage.

### Checkpoint, and prove rollback works

- Before every round: `end(True)` → `save()` → snapshot → version-control step. Ordering
  matters — a snapshot taken before a pending save silently captures the *previous* state
  while claiming to be current.
- A script can check in: `end(True)` → `currentProgram.save(comment, monitor)` →
  `domainFile.checkin(handler, monitor)`. "File has unsaved changes" is literal: save.
- **Exercise a restore before you need one.** Passing an integrity check is not a proven
  rollback path.

## What actually improves decompilation

- **Function signatures.** Parameter/return types propagate to call sites — the strongest
  real cascade. Mangled names encode full signatures;
  `DemanglerCmd(addr, mangled).applyTo(program, monitor)` applies name *and* signature.
- **Defining functions at undefined code.** Feeds analyzers new material, extends
  pointer-table runs truncated by undefined targets, adds call-graph edges. Often the
  highest-leverage single mutation.
- **Struct layouts.** Field accesses become named. Can *change* signatures as a side
  effect: a large struct returned by value switches to the hidden return-storage-pointer
  convention, so `T Func(this)` becomes `T * Func(this, T *__return_storage_ptr__)`.
- **Data mutability.** *"The decompiler will display the contents of a memory location if
  the contents are marked as constant. Otherwise it will display a pointer to the
  location."* Settings: normal / constant / volatile / writable — per-data, or
  per-memory-block via the Memory Map. Marking genuinely read-only sections constant is
  factually correct, cheap, and high-leverage. Marking hardware registers **volatile**
  stops the decompiler folding away repeated reads.

## Harvesting traps

- **The decompiler's signature is not necessarily the database's.** With no committed
  signature the decompiler applies local heuristics, so its display can differ from the
  Listing and *two call sites of the same function can show different signatures*.
  Reading a signature from decompiler output is not evidence it is in the database.
  `Commit Params/Return` commits it.
- **Harvesters are not idempotent across an apply.** The same harvester can legitimately
  yield different evidence afterward — a matcher for `T Func(...)` stops matching once an
  apply rewrites it. So: **stamp every evidence row with the program version it came
  from**, treat earlier evidence as **append-only**, and teach post-apply harvesters to
  accept the post-apply shape. Regenerating after an apply can silently drop whole
  witness categories while looking like a clean re-run.
- **Distinguish measured-zero from structural-zero.** "I looked and found nothing" and
  "there was nothing to look at" are different findings. Emit counters that separate them.
- **A short name is not an identity.** Demangled short names collide across a hierarchy —
  an override shares its base's name, so two different tables can look identical. Join on
  **addresses**, not names.
- **`Function.getName()` is unqualified; `getName(True)` includes the namespace.**
  Comparing the wrong one against a namespace-qualified expectation makes a check
  *structurally unable to match* — it reports 0/N forever and reads as a failing sweep
  rather than a broken test. This exact bug produced four dead checks in this project.
  Likewise raw mangled names and demangled short names are different string spaces that
  never intersect; converting one to the other is required before comparison.

## Assertion discipline under iteration

- **Prefer checks that already fail.** Where binary ground truth contradicts current
  state, those disagreements are free demonstrations that a check fires. Cheaper and
  stronger than constructed poison.
- **Poison where `expected` and `actual` might share a source** — the classic dead check
  is two reads of the same variable in the same loop compared to each other.
- **Never verify against state you produced this round.** The `SourceType.AI` filter
  enforces this for harvesters; apply it by hand elsewhere.
- **Never verify against an artifact measured wrong**, including its unchecked parts.
- Test verification logic against poisoned *copies of files*, never the live program.

---

# Part 2 — Game reverse engineering

Games differ from other targets in three useful ways: they leak an unusual amount of
developer-facing text, they have a large body of external documentation written by
players and modders, and **you can make them change state on demand**, which is a search
primitive static analysis does not have.

## Sweep for free names before analyzing anything

Ordered by payoff. Do these before you interpret a single function — every name found
here is `IMPORTED`-grade, and analysis done without them is wasted effort.

1. **Assert / debug strings containing source paths.** `__FILE__` in an assert macro
   embeds the **original source file path**, often with line numbers. This reconstructs
   the developer's module decomposition — directory names are subsystems, filenames are
   translation units — and it is the single highest-value string sweep in game RE. A
   function containing `"c:\\proj\\ai\\pathfind.cpp"` belongs to the pathfinding module,
   whatever it is called.
2. **Exported and mangled symbols; leftover PDB, MAP, DWARF, or `.debug` data.** Mangled
   C++ names carry class, member, virtualness, and full signature. Even a stripped retail
   build often exports more than expected.
3. **Other builds of the same game.** Demos, betas, console/other-platform ports,
   patched versions, and debug builds frequently ship symbols the retail build stripped.
   Diffing builds also isolates what a patch changed. Ghidra's **Version Tracking** and
   **BSim** exist for exactly this correlation.
4. **Embedded scripting VMs.** Lua and custom interpreters register native functions by
   **name** in a table — a registration table is a free symbol table mapping strings to
   function pointers. Find the registrar, get hundreds of names at once.
5. **Config / INI / CVar / console-command parsers.** A string→offset or string→setter
   table names struct fields and globals directly, in the developers' own vocabulary.
6. **Library and engine fingerprints.** Version strings, banners, and known code
   (Miles Sound System, Bink, RenderWare, Glide, DirectX, Havok, zlib, id/Build/Unreal
   lineage). Identifying an engine or middleware imports an entire published API surface
   at once. Ghidra's **Function ID (FID)** does byte-level library matching — but its
   results are **lower bounds**, silent on code that neither calls marker imports nor
   matches the corpus.
7. **Localization tables, asset and archive filenames, resource IDs.** These name game
   *concepts*, which is what you want for struct and enum naming.
8. **RTTI and vtable symbols**, where the language and build produced them
   (MSVC `??_7Class@@6B@` vftable symbols, `.?AV...@@` type descriptors, Itanium-ABI
   `_ZTV`). Absence is a measured finding — a build with `/GR-` has none, and that closes
   the cheapest naming route rather than meaning "look harder."

**Use the developers' vocabulary.** Terminology from the game's own UI, manual, and
strings is what the authors called these things. Name recovered entities to match it, not
to match your model of the game.

## External corroboration is legitimate evidence — cite it as such

Modding communities, file-format wikis, existing mod tools and editors, published SDKs,
strategy guides, and shipped source for the same engine are all real sources. They are
*not* binary ground truth: record them at their own provenance level, never as
`IMPORTED`. A format documented by a modder and confirmed against the bytes is strong; a
format documented by a modder and merely *assumed* is a hypothesis.

## Dispatch recovery — virtual calls are one case of several

Do not force an OOP reading onto a program that isn't OOP. Era matters: pre-C++ and
early-C++ game code dispatches through plain function-pointer tables far more often than
through vtables, and the recovery mechanics are the same while the *semantics* are not.

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

### If it *is* C++, and you want names in the decompiler

**First: `ghidra.program.model.gclass.ClassUtils` exists.** Ghidra has a built-in
convention for modelling C++ classes with vtables — `isVTable()`,
`getVftDefaultEntry()`/`getVftEntrySize()`, `getVbtDefaultEntry()` for *virtual base*
tables under virtual inheritance, `getClassInternalsPath()` for its `_internals` category
layout, `getSelfBaseType()`, `getBaseClassDataTypePath()`, `getReplacementPointers()`.
Follow it and your results interoperate with Ghidra's PDB/RTTI machinery; hand-roll and
they sit parallel to it. Check `ClassUtils` before building any vftable struct by hand.

Ghidra does not devirtualize automatically — it cannot prove a vtable pointer is constant
after assignment (issues #650, #516). Three explicit mechanisms, escalating:

1. **Name the slots** (Ghidra's official recipe): a `FunctionDefinition` data type per
   virtual method with `__thiscall`; a `<Class>_vftable` struct whose fields are those
   definitions **in slot order**, each **named** after its method; the class struct's
   first field a pointer to that vftable struct.
2. **Mark the vtable data constant** so the decompiler reads *through* it instead of
   showing a pointer to it.
3. **Override the call site** — `RefType.CALL_OVERRIDE_UNCONDITIONAL` (also
   `JUMP_OVERRIDE_UNCONDITIONAL`, `CALLOTHER_OVERRIDE_CALL/JUMP`) turns an indirect call
   into a direct one. This is how Ghidra 12.1 cleaned up Objective-C `_objc_msgSend`.

Mechanism 3 writes *your inference into the call graph*. Tag it, record it, and never let
a harvest read it back as fact.

## Data shapes characteristic of games

- **Fixed-point arithmetic.** On pre-FPU and console targets, an integer multiply
  followed by a shift is Q16.16-style fixed point, not a bug and not an int. Ghidra will
  show no float type; the shift amount tells you the binary point.
- **FPU era.** x87 stack code decompiles awkwardly and its register-stack discipline
  matters for reading arguments; SSE-era code looks entirely different for the same math.
- **Entity pools and handles.** Fixed-size object arrays with active flags or freelists,
  and opaque handle types (often `index | generation<<n`) rather than pointers. A 4-byte
  "object id" scalar passed everywhere is a handle, and its bit split is worth recovering.
- **Struct-of-arrays vs array-of-structs.** SoA layouts break the "one struct per
  object" assumption — parallel arrays indexed by entity id look like unrelated globals.
- **Custom allocators / arenas.** Games frequently avoid `operator new` per object, so
  allocation-size evidence for a type may be genuinely absent rather than missed. Stack
  and embedded value types often have no heap allocation site at all.
- **Data-driven tuning tables.** Arrays of structs in read-only data holding balance,
  unit, and weapon stats. Recovering the struct recovers the game's design data, and the
  table's field count and stride are strong layout evidence.
- **Save files and network packets are serialized structs** — a Rosetta stone for
  layout. Multiple saves differing in one known way isolate one field. Look for something
  constant in size and structure with changing contents; a value the player can change on
  demand is a labelled sample.
- **Update loops** with a large switch on a state enum, called once per tick.
- **Packed, encrypted, or self-modifying code** in copy protection; overlays and
  bank-switching (see below) can also make one address hold different code over time.

## Platform breadth

Ghidra supports far more than PE/x86, and console targets are common in game RE: MIPS
(PS1/PS2/N64/PSP), PowerPC (GameCube/Wii/Xbox 360/PS3), SH-2 (Saturn), 68k
(Genesis/Amiga/Mac), ARM (GBA/DS/mobile), x86 (DOS/Windows).

- **Register context matters for correct disassembly.** MIPS `$gp`-relative addressing,
  ARM/Thumb mode selection, and similar per-region processor state must be set or whole
  regions disassemble as garbage. Set register values over an address range rather than
  fighting the output.
- **Overlays.** PS1 and DOS games routinely load different code to the same address.
  Model these as Ghidra **overlay memory blocks**, or analysis of one overlay silently
  corrupts another. An address alone is not an identity in an overlaid program.
- **Bank switching** on cartridge systems has the same consequence.
- **Memory-mapped hardware registers.** Label them from platform documentation and mark
  them **volatile**; a write to a known VDP/GPU/DMA register identifies surrounding code
  instantly, and volatility stops the decompiler optimizing repeated accesses away.
- **Loaders.** Console ROM and executable formats often need a community loader
  extension; failing that, import raw with a correct base address and memory map.

## Runtime observation — the primitive games give you for free

You can *cause* a change: take damage, move, spend currency, open a menu. That converts
"what is this field" into a controlled experiment.

- **Known-value memory search** (the Cheat Engine pattern): find the address holding a
  value you can change, then xref it back to code. This finds structs static analysis
  cannot reach.
- **Breakpoint on a known event** to catch the `this` pointer of an object whose identity
  you already know, then dump its bytes across several instances.
- Ghidra's own **Debugger** connects to a live process; its **p-code Emulator**
  (`ghidra.app.emulator.EmulatorHelper`) runs code *without* the game — ideal for
  decompression, checksum, and decryption routines where you want the algorithm's output
  rather than a live session.
- Runtime is the right instrument when static evidence is *exhausted*, not merely
  inconvenient. Phrase the question as a specific breakpoint and byte range before
  reaching for it.

---

# Part 3 — Tool surface

## Cascade triggers, scoped

Auto-analysis is `ENABLED` by default during scripts ("Auto-Analysis responding to
changes"), so mutations can trigger analysis *while your loop runs* and you may read
half-analyzed state. Mutate first, analyze deliberately.

- `analyzeChanges(program)` — "Starts auto-analysis if not started and waits for pending
  analysis to complete." **The sanctioned trigger.**
- `analyzeAll(program)` — full re-analysis. Reckless on a curated program; can undo
  curation. Avoid.
- Surgical: `AutoAnalysisManager.getAnalysisManager(program)` exposes
  `codeDefined(addr|set)`, `functionDefined(addr|set)`,
  `functionSignatureChanged(addr|set)`, `dataDefined(set)`,
  `createFunction(target, findFunctionStart)`, `getAnalyzer(name)`,
  `cancelQueuedTasks()`. Tell it what changed, then `analyzeChanges`. Not in the public
  javadoc — reachable but unsupported.
- `GhidraScript.AnalysisMode`: `ENABLED` (live), `SUSPENDED` (queued until the script
  ends), `DISABLED` (change events ignored).

## Check for a built-in before writing your own

We wrote a vtable sweeper from scratch without checking that the first item existed.

| Need | Built-in |
|---|---|
| Find function-pointer / vtable tables | `Search → For Address Tables` |
| Model a C++ class with vtables | `ClassUtils` / `ClassID` (`ghidra.program.model.gclass`) — a whole convention, don't hand-roll |
| MSVC RTTI structures | `RTTI0DataType`…`RTTI4DataType`, `MSDataTypeUtils` (`ghidra.app.util.datatype.microsoft`) |
| Infer struct fields from access patterns | Auto Create / Auto Fill in Structure; **Auto Fill in Class Structure** for a known `this` |
| Set data constant / volatile | `MutabilitySettingsDefinition`: `NORMAL`/`CONSTANT`/`VOLATILE`/`WRITABLE` |
| Parse C headers into the program | `CParserUtils.parseHeaderFiles(...)` (C only, not C++) |
| Open a `.gdt` type archive with no tool | `FileDataTypeManager.openFileArchive(file, false)` |
| **Decompile many functions** | `ParallelDecompiler.decompileFunctions(...)` — never loop `DecompInterface` serially over a whole program |
| **Record `__FILE__` / source-path evidence** | `program.getSourceFileManager().addSourceMapEntry(file, line, range)` — queryable both ways, not a comment |
| Custom function-start signatures, library fingerprints | `ghidra.util.bytesearch`: `DittedBitSequence`, `PatternPairSet`, `MemoryBytePatternSearcher` |
| Find more functions in undefined bytes | `RandomForestFunctionFinderPlugin` (MachineLearning extension) — trains on functions already found |
| Identify library code | Function ID (FID); **BSim** for similarity search across corpora |
| Compare two builds / match stripped against symbolled | Version Tracking, BSim, `ghidra.program.model.correlate` (`HashedFunctionAddressCorrelation`, `MnemonicHashCalculator`) |
| Unrecovered switch / jumptable | `FindUnrecoveredSwitchesScript.java`, `SwitchOverride.java` |
| Non-returning-function damage | `FixupNoReturnFunctionsScript.java` |
| Run a routine without the game | `EmulatorHelper` (`ghidra.app.emulator`) |
| **Enforce a non-mutating pass** | `analyzeHeadless -readOnly` — makes evidence-only a harness guarantee, not a discipline |
| Batch/pipeline runs | `analyzeHeadless -process -preScript -postScript -noanalysis` |
| Preview bytes without committing | Disassembled View window |

## Operating rules (MCP)

- **Editing a script in your repo does nothing.** Re-install into Ghidra's user script
  directory before every run (`action="create"`, `overwrite=true`) — **library modules
  included**: Ghidra's script dir sits earlier on `sys.path` than an appended repo path,
  so a stale installed copy shadows your edit silently and you debug logic that is not
  running.
- **`action="run"` is asynchronous.** It returns a task id; nothing is verified until you
  read `get_task_status`. Never report a result you have not read.
- A `create` response may misreport the provider (`<unsupported>`, `Jython`). Cosmetic —
  the `run` result reports the real one.
- Prefer one script run over hundreds of per-item MCP calls; and only a script can set
  `SourceType`.

## Scripting rules (PyGhidra)

- **CPython 3 via PyGhidra**, not Jython — `except X as e`, f-strings, real imports. As
  of Ghidra 12.1 Jython is an opt-in extension, so a `# @runtime Jython` tag fails before
  the first line unless installed. Ghidra 12.1 needs JDK 21+.
- Wrap DB modification in a transaction; `GhidraScript` opens one and `end(True)` closes
  it (required before save/checkin).
- Dispose decompilers and close files explicitly; pass `monitor` to long operations.
- **Discover API from the local install, not memory or the web.** A Ghidra install ships
  `docs/ghidra_stubs/pypredef/` (hundreds of greppable, exact-version stub files covering
  classes the public javadoc omits, e.g. `AutoAnalysisManager`),
  `docs/GhidraAPI_javadoc.zip`, `docs/CheatSheet.html`, `docs/ChangeHistory.md`, and
  `docs/GhidraClass/` course material — of which
  `Advanced/improvingDisassemblyAndDecompilation.pdf` is the authoritative guide to most
  of Part 1. Search these with Python if your shell's text tools are proxied or unreliable.
- Ghidra writes the host filesystem with plain `open()`. Windows-hosted Ghidra driven from
  WSL: script source needs Windows paths (`C:\...`); read the same files at `/mnt/c/...`.

## Failure modes to design against

- **Metric gaming.** Given one score to optimize, agents optimize the score. In one study
  they found that *minimizing changes* maximized a binary-similarity metric and reverted
  variables to `rax`/`rcx` — the measure became the target and stopped measuring. Use
  multi-dimensional criteria; never let "verification passed" be the objective.
- **Coverage theatre.** The same study: 10–15% of functions genuinely analyzed while the
  run reported completion. State coverage as a number, always.
- **Context rot** across long runs, producing inconsistent naming between functions
  analyzed early and late. Short scoped executions beat one monolithic session.
- **A reproducible error is not evidence for your theory about its cause.** It is evidence
  the same trigger fires every time. Read the error text literally first — "file has
  unsaved changes" means the file has unsaved changes.
- **Volume is not the metric.** A confidently wrong result is worse than an absent one,
  because later rounds treat it as evidence.

## Sources

Ghidra 12.1.2 local install: `docs/GhidraClass/Advanced/improvingDisassemblyAndDecompilation.pdf`
(vftable recipe, data mutability, signature commit semantics, Decompiler Parameter ID
warning, non-returning functions, switch/flow overrides), `docs/WhatsNew.md` (12.1
bitfield recovery, Objective-C call overriding, Jython-as-extension, JDK 21),
`docs/ghidra_stubs/pypredef/` (`AutoAnalysisManager`, `RefType`, `SourceType`,
`EmulatorHelper`, BSim packages), `docs/GhidraAPI_javadoc.zip`.

- [SourceType javadoc](https://ghidra.re/ghidra_docs/api/ghidra/program/model/symbol/SourceType.html) · [GhidraScript.AnalysisMode](http://ghidra.re/ghidra_docs/api/ghidra/app/script/GhidraScript.AnalysisMode.html)
- [Ghidra #650](https://github.com/NationalSecurityAgency/ghidra/issues/650), [#516](https://github.com/NationalSecurityAgency/ghidra/issues/516) — no automatic devirtualization
- [Override Call Reference](https://grant-h.github.io/docs/ghidra/decompiler/classOverride.html) · [decompiler override.cc](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/Decompiler/src/decompile/cpp/override.cc)
- [DemanglerCmd source](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/Base/src/main/java/ghidra/app/cmd/label/DemanglerCmd.java)
- [GhidrAssistMCP source](https://github.com/jtang613/GhidrAssistMCP) — audited for `SourceType` usage
- [LLM Agent-Assisted RE with Quantitative Readability Metrics](https://arxiv.org/html/2606.06838) — metric gaming, coverage, context rot
- [Agentic RE: Building Custom AI Skills with Coding Agents, Recon 2026](https://cfp.recon.cx/recon-2026/talk/SHYHKM/)
- [und3rf10w ghidra-scripting agent skill](https://skillsmp.com/creators/und3rf10w/ai-ghidra-tools/plugins-ghidra-skills-ghidra-scripting)
- [Beginners Guide to Reverse Engineering (Retro Games)](https://www.retroreversing.com/tutorials/introduction) · [Reverse Engineering a GBA Game](https://www.starcubelabs.com/reverse-engineering-gba/) · [Reverse engineering save game files](https://mytechnologyblog532.wordpress.com/2016/11/13/reverse-engineering-save-game-files/) · [Unity IL2CPP save file RE](https://blog.painite.ch/en/blog/unity-save-file-reverse-engineering/)
- This project: `notes/tooling-capabilities.md`, `notes/assertion-discipline.md`,
  `notes/phase2-report.md` §5
