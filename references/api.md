# Ghidra API reference for iterative RE

Verified against a **Ghidra 12.1.2** install by reading
`docs/ghidra_stubs/pypredef/` and `docs/GhidraAPI_javadoc.zip`. Signatures are
paraphrased for PyGhidra (CPython 3 via JPype). Anything marked *(doc)* is cited from
Ghidra's own documentation but not executed here.

## How to discover API — do this instead of recalling it

A Ghidra install carries a complete, exact-version API surface offline:

| Path | What it is |
|---|---|
| `docs/ghidra_stubs/pypredef/` | **777 package stub files.** Greppable, one per package, includes classes the public javadoc omits (e.g. `AutoAnalysisManager`). Best discovery tool. |
| `docs/ghidra_stubs/typestubs/`, `ghidra_stubs-*.whl` | Same content as installable type stubs |
| `docs/GhidraAPI_javadoc.zip` | Full javadoc (51 MB) — has the prose descriptions the stubs lack |
| `docs/GhidraClass/Advanced/improvingDisassemblyAndDecompilation.pdf` | Authoritative guide to improving decompilation |
| `docs/GhidraClass/{Beginner,Intermediate,AdvancedDevelopment,BSim,Debugger}/` | Course material; `Debugger/B2-Emulation`, `Debugger/B3-Scripting` are substantial |
| `support/analyzeHeadlessREADME.md` | Headless automation reference |
| `docs/CheatSheet.html`, `docs/ChangeHistory.md`, `docs/WhatsNew.md` | UI actions, version deltas |
| `docs/README_PDB.html` | PDB parser — read this **first** on any Windows target, before assuming symbols are gone |
| `docs/languages/` | Processor/Sleigh docs |
| **`Ghidra/*/*/lib/*-src.zip`** | **The Java SOURCE, per module.** Not one archive — one zip beside each jar. This is the only place that answers *"which arm of this call mutates"*, which neither the stubs nor the javadoc state. |

Search these with Python, not shell text tools, if your environment proxies `grep`.

**Read the source, not just the stubs, before depending on a shipped helper.** Measured: three
questions this reference now answers — which `FillOutStructureHelper` argument mutates, why a
shipped byte pattern never fires, and what `checkAfterName` accepts — were **unanswerable from the
javadoc** and took one read each from the source zips. Finding the right zip:

```bash
find "$GHIDRA" -name '*-src.zip' -print0 | while IFS= read -r -d '' z; do
  unzip -l "$z" 2>/dev/null | grep -q 'TheClass.java' && echo "$z"
done
```

Note the `-print0`/`read -d ''`: under **zsh** an unquoted `$(find ...)` in a `for` loop does
**not** word-split, so the naive form silently hands every path to one `unzip` and reports
nothing. An empty search result is a claim about your search until the search is shown to work.
Useful locations, verified on 12.1.2 — the module a class lives in is rarely the one you guess:

| Class | Module |
|---|---|
| `ghidra.util.bytesearch.*` | `Ghidra/Features/Base/lib/Base-src.zip` |
| `ghidra.app.analyzers.FunctionStartAnalyzer`, `Patterns` | `Ghidra/Features/**BytePatterns**/lib/BytePatterns-src.zip` |
| `ghidra.app.decompiler.util.FillOutStructureHelper` | `Ghidra/Features/Decompiler/lib/Decompiler-src.zip` |
| `ghidra.program.model.data.NoisyStructureBuilder` | `Ghidra/Framework/SoftwareModeling/lib/SoftwareModeling-src.zip` |

## The MCP boundary — what needs a script

An MCP server gives you interactive **reads** cheaply and is the wrong instrument for everything
below. Reach for a script when you need any of these; none is reachable through a read-only tool
surface, and several are actively unsafe through a mutating one.

| Need | Why MCP cannot | See |
|---|---|---|
| Set `SourceType` on anything | Mutating MCP tools hardcode `USER_DEFINED` — audited, 18 occurrences | SKILL.md, "Mutate through scripts" |
| Any bracketed mutation | The invariant diff has to wrap the operation in one transaction | SKILL.md, "Mutation safety" |
| Bulk p-code / SSA work | Per-item calls; you want `ParallelDecompiler` or one `DecompInterface` | "Decompiler" below |
| Drive a shipped analyzer's machinery yourself | No tool exposes `PatternFactory`, `MatchAction`, `PseudoDisassembler` | "Byte pattern search", "Read-only witnesses" |
| Read raw instruction p-code | `Instruction.getPcode()` is the SLEIGH spec; no MCP equivalent | "Program, listing, functions" |
| Enumerate undisassembled ranges | `Listing.getUndefinedRanges` over `Memory.getExecuteSet()` | "Program, listing, functions" |
| Walk the program's version history | `getVersionHistory()` / `getReadOnlyDomainObject` | "Version control and checkpointing" |
| Anything needing a Java interface implementation | `@JImplements` from PyGhidra | "PyGhidra / JPype interop" |

The converse is worth stating too: **MCP is right for orientation** — `get_binary_info`,
`analysis_options` (which analyzers ran, and at what settings), `get_strings`, `xrefs`,
`get_function_statistics`. One `analysis_options` call answered "were the function-start
analyzers even enabled" in seconds, and that is a fact no script should be written to obtain.

## Provenance — `ghidra.program.model.symbol.SourceType`

Constants: `USER_DEFINED`, `IMPORTED`, `ANALYSIS`, `AI`, `DEFAULT`.
Priority: `USER_DEFINED` > `IMPORTED` > `ANALYSIS` = `AI` > `DEFAULT`.
Methods: `getPriority()`, `isHigherPriorityThan(other)`, `isLowerPriorityThan(other)`.

`AI` is documented as *"content produced through AI assistance."* Read a symbol's tier
with `symbol.getSource()`. **Tag every agent mutation `SourceType.AI`; filter it out of
every harvest.**

## Program, listing, functions

```python
currentProgram.getFunctionManager()      # .getFunctions(True), .getExternalFunctions()
currentProgram.getSymbolTable()
currentProgram.getDataTypeManager()
currentProgram.getListing()
currentProgram.getMemory()
currentProgram.getReferenceManager()
currentProgram.getBookmarkManager()
currentProgram.getSourceFileManager()
```

`FlatProgramAPI` / `GhidraScript` conveniences: `toAddr(x)`, `getFunctionAt(addr)`,
`getFunctionContaining(addr)`, `getFunction(name)`, `getInstructionAt(addr)`,
`getDataAt(addr)`, `getBytes(addr, n)`, `getReferencesTo(addr)`,
`getReferencesFrom(addr)`, `getSymbolAt(addr)`, `createLabel(addr, name, makePrimary)`,
`getScriptArgs()`, `createFunction(entry, name)`, `disassemble(addr)`.

**Count functions consistently.** `getFunctionCount()` **includes** external functions;
`getFunctions(boolean)` returns **non-external** functions only. They differ, so record
which you used — and note two things the obvious reading gets wrong: the boolean is
**direction** (`true` = ascending address order), *not* a filter, and an internal
thunk-to-DLL (a `JMP [IAT]` stub with an entry point in `.text`) **is** returned by it.

## Symbols, namespaces, classes

```python
st.createLabel(addr, name, SourceType.AI)
st.createClass(parentNamespace, name, SourceType.AI)      # a GhidraClass
st.createNameSpace(parentNamespace, name, SourceType.AI)
sym.setName(name, SourceType.AI)
sym.getSource(); sym.getSymbolType(); sym.getParentNamespace()
func.setName(name, SourceType.AI)
func.setParentNamespace(ns)
```

`Function.getName()` returns the **unqualified** name; `getName(True)` includes the
namespace. Comparing the wrong one against a namespace-qualified expectation makes a
check structurally unable to match — a documented repeat failure in this project.

## Data types and structures

`ghidra.program.model.data`:
`StructureDataType`, `UnionDataType`, `EnumDataType`, `TypedefDataType`,
`ArrayDataType`, `PointerDataType`, `FunctionDefinitionDataType`,
`CategoryPath`, `DataTypePath`, `DataTypeConflictHandler`.

```python
s = StructureDataType(CategoryPath("/Game"), "CThing", 0, dtm)
s.add(dt, length, name, comment)
s.insertAtOffset(offset, dt, length, name, comment)
s.replaceAtOffset(offset, dt, length, name, comment)
dtm.addDataType(s, DataTypeConflictHandler.REPLACE_HANDLER)
```

**Mutability** — `MutabilitySettingsDefinition` with constants `NORMAL`, `VOLATILE`,
`CONSTANT`, `WRITABLE`; applied through a `Settings` object on defined data (or per
memory block via the Memory Map). *Marking data `CONSTANT` makes the decompiler show
its contents rather than a pointer to it* — the mechanism for reading through a vtable
or a read-only table. `VOLATILE` stops the decompiler folding repeated hardware-register
reads.

### C++ classes: use Ghidra's own conventions, not hand-rolled structs

`ghidra.program.model.gclass` provides `ClassID` and `ClassUtils`, which encode Ghidra's
convention for modelling C++ classes with vtables. **Check these before hand-building
`<Class>_vftable` structs:**

```python
ClassUtils.isVTable(dataType)
ClassUtils.getVftDefaultEntry(dtm)        # PointerDataType for a vftable slot
ClassUtils.getVftEntrySize(dtm)
ClassUtils.getVbtDefaultEntry(dtm)        # virtual BASE table (virtual inheritance)
ClassUtils.getVbtEntrySize(dtm)
ClassUtils.getClassPath(classId)
ClassUtils.getClassInternalsPath(composite)   # category path for "class internals"
ClassUtils.getSelfBaseType(composite)
ClassUtils.getBaseClassDataTypePath(composite)
ClassUtils.getReplacementPointers(dtm, structure)
ClassUtils.hasClassAttribute(structure)
ClassUtils.createVxTableDescriptionOffsetTag(ptrOffsetInClass)
```

Following these keeps results interoperable with Ghidra's PDB and RTTI machinery instead
of parallel to it.

MSVC RTTI models live in `ghidra.app.util.datatype.microsoft`: `RTTI0DataType` …
`RTTI4DataType`, `MSDataTypeUtils`, `GuidUtil`, `ThreadEnvironmentBlock`. Useful when the
target *has* RTTI; absence (no `.?AV` / `type_info` strings) is a measured finding.

## Function signatures

```python
from ghidra.app.cmd.function import ApplyFunctionSignatureCmd
ApplyFunctionSignatureCmd(entryPoint, signature, SourceType.AI).applyTo(program, monitor)
```

Also in `ghidra.app.cmd.function` (73K of commands): `CreateFunctionCmd`,
`DeleteFunctionCmd`, `SetFunctionNameCmd`, `AddFunctionTagCmd`,
`SetReturnDataTypeCmd`, `SetVariableNameCmd`, `CaptureFunctionDataTypesCmd`,
`ApplyFunctionDataTypesCmd`.

> **`ApplyFunctionDataTypesCmd` is dangerous with a broad address set.** Recorded
> incident: with the whole initialized address set and a large archive it silently
> destroyed two unrelated functions — clean return, no exception. Prefer per-function
> `ApplyFunctionSignatureCmd` over a broad archive apply, and always bracket with an
> invariant diff.

**Listing vs decompiler signature** *(doc)*: with no committed signature the decompiler
synthesizes one from local heuristics, so its display can differ from the Listing and two
call sites of one function can show different signatures. `Commit Params/Return` commits
it. Reading a signature from decompiler output is *not* evidence it is in the database.

## References and overrides

`RefType` constants relevant to overriding control flow:
`CALL_OVERRIDE_UNCONDITIONAL`, `JUMP_OVERRIDE_UNCONDITIONAL`,
`CALLOTHER_OVERRIDE_CALL`, `CALLOTHER_OVERRIDE_JUMP`, plus `COMPUTED_CALL`,
`COMPUTED_JUMP`, `INDIRECTION`.

Adding a primary reference of the appropriate override type at a call/jump site converts an
indirect transfer into a direct one for the decompiler.

**Scope note.** Of the three mechanisms for making virtual dispatch readable (see
`cpp-abi.md`), only this one changes what the *call* is: typing the vtable and marking it
constant make the table and its slots **legible**, but the call site remains indirect and no
call-graph edge appears. So this is the only route to true devirtualization, while the other
two are usually enough if you only need readable output. Decide which you actually need —
this one writes *your inference into the call graph*, so tag it `SourceType.AI` and never
harvest it back as fact.

Commands in `ghidra.app.cmd.refs`: `AddMemRefsCmd`, `AddOffsetMemRefCmd`,
`AddShiftedMemRefCmd`, `AddRegisterRefCmd`, `AddStackRefCmd`, `EditRefTypeCmd`,
`RemoveReferenceCmd`, `AssociateSymbolCmd`, `SetFallThroughCmd`, `ClearFallThroughCmd`.

## Analysis control

```python
analyzeChanges(currentProgram)   # GhidraScript: start analysis, wait for pending
analyzeAll(currentProgram)       # full re-analysis -- avoid on a curated program
```

`GhidraScript.AnalysisMode`: `ENABLED` (default, "Auto-Analysis responding to changes"),
`SUSPENDED` (change events analyzed after the script ends), `DISABLED` (change events
ignored). Both non-default modes wait for pending analysis first.

Surgical, via `AutoAnalysisManager.getAnalysisManager(program)` — **not in the public
javadoc; reachable but unsupported**:

```python
codeDefined(addr | addressSet)
dataDefined(addressSet)
functionDefined(addr | addressSet)
functionModifierChanged(addr | addressSet)
functionSignatureChanged(addr | addressSet)
createFunction(target, findFunctionStart)
createFunction(targetSet, findFunctionStarts, priority)
getAnalyzer(name); cancelQueuedTasks(); getAnalysisTool()
```

Tell it precisely what changed, then `analyzeChanges` — not `analyzeAll`.

Ghidra's course warns of the **Decompiler Parameter ID** analyzer: run it "too early or
before fixing problems" and you "propagate bad information all over the program." *(doc)*

## Decompiler

```python
from ghidra.app.decompiler import DecompInterface, DecompileOptions
ifc = DecompInterface(); ifc.openProgram(currentProgram)
res = ifc.decompileFunction(func, timeoutSecs, monitor)
res.getDecompiledFunction().getC(); res.getHighFunction()
ifc.dispose()          # always
```

**Batch decompilation** — `ghidra.app.decompiler.parallel`:
`ParallelDecompiler.decompileFunctions(callback, program, functions, monitor)`,
`createChunkingParallelDecompiler(callback, monitor)`, plus `DecompilerCallback`,
`DecompileConfigurer`. Use this for whole-program sweeps rather than looping
`DecompInterface` serially.

`ghidra.program.model.pcode`: `HighFunction`, `HighSymbol`, `HighFunctionDBUtil`
(`updateDBVariable(...)` commits decompiler-level variables to the database),
`PcodeOp`, `Varnode`.

For `this`-relative offset recovery over SSA, do not hand-roll a register tracker — see
`FillOutStructureHelper` under "Read-only witnesses", including which of its arms writes to the
DataTypeManager and why its reach dies on a `__cdecl` prototype.

## Demangling

```python
from ghidra.app.cmd.label import DemanglerCmd
DemanglerCmd(addr, mangledName).applyTo(program, monitor)   # applies NAME and SIGNATURE
```

`ghidra.app.util.demangler` (75K): `DemangledFunction`, `DemangledObject`,
`DemanglerOptions`, `DemanglerUtil`; MSVC specifics in
`ghidra.app.util.demangler.microsoft`. Ghidra 12.1 added Microsoft demangler **output
options** controlling user-defined-type tags and anonymous-namespace naming *(doc)*.
Rust demangling: `ghidra.app.plugin.core.analysis.rust.demangler`.

MSVC mangled names encode the declaring class, the access/virtual specifier (the character
after `@@` — `Q` public non-virtual, `U` public virtual, `M` protected virtual,
`S` public static, `E` private virtual), and the full parameter list. `??_7Class@@6B@` is a
**vftable** symbol whose address is the class's vtable.

## Source map — the native home for `__FILE__` evidence

`currentProgram.getSourceFileManager()` (`ghidra.program.model.sourcemap`):

```python
sfm.addSourceFile(sourceFile)
sfm.addSourceMapEntry(sourceFile, lineNumber, addressRange)
sfm.addSourceMapEntry(sourceFile, lineNumber, baseAddr, length)
sfm.getSourceMapEntries(addr)                      # what source is this address from?
sfm.getSourceMapEntries(sourceFile)                # every address from this file
sfm.getSourceMapEntries(sourceFile, minLine, maxLine)
sfm.getAllSourceFiles(); sfm.getMappedSourceFiles()
```

When assert strings leak original source paths, record them as **real source map
entries** rather than comments. The mapping is then queryable in both directions, which
effectively reconstructs the original translation units as first-class program data.

## Byte pattern search

`ghidra.util.bytesearch`: `DittedBitSequence` (wildcarded patterns), `Pattern`,
`PatternPairSet` (pre/post patterns — the function-start-pattern mechanism),
`MemoryBytePatternSearcher`, `ProgramMemorySearcher`, `BulkPatternSearcher`,
`GenericMatchAction`, `DummyMatchAction`, `AlignRule`, `PostRule`, `Match`,
`PatternFactory`, `MatchAction`. Also `ghidra.features.base.memsearch.*` for the
memory-search API.

Use for custom function-start signatures (attacking undefined-code backlogs) and library
fingerprinting — **and to ask what the shipped function-start analyzer would do, without
running it.** The whole pipeline is drivable from a script, which is strictly better than
reimplementing ditted-bit matching:

```python
from ghidra.app.analyzers import Patterns              # Features/BytePatterns
from ghidra.util.bytesearch import Pattern, MemoryBytePatternSearcher
from java.util import ArrayList

tree  = Patterns.getPatternDecisionTree()               # patternconstraints.xml
Patterns.hasPatternFiles(program, tree)                 # bool
files = Patterns.findPatternFiles(program, tree)        # ResourceFile[] — ASK, don't assume
pats  = ArrayList()
for f in files:
    Pattern.readPatterns(f, pats, factory)              # factory: your PatternFactory
searcher = MemoryBytePatternSearcher("probe", pats)
searcher.setSearchExecutableOnly(True)
searcher.searchAll(program, monitor)                    # or .search(program, addressSet, monitor)
```

Four things the javadoc does not tell you, all read from the source and all load-bearing:

- **A pattern file is a cross product filtered by an information budget, not a list of byte
  strings.** `PatternPairSet.createFinalPatterns` pairs each prepattern with each postpattern
  and keeps the concatenation only if `postcheck >= postBitsOfCheck` **and**
  `precheck + postcheck >= totalBitsOfCheck`, counting **fixed (non-ditted) bits**. With
  `x86win_patterns.xml`'s `totalbits="32" postbits="16"`, the pair `0x90` (8 fixed bits) with
  `0x83ec 0.....00` (19) totals 27 and **is never built** — so a prologue that visibly matches
  can be one the engine never had a pattern for. `Pattern.getMarkOffset()` is the prepattern
  length, i.e. the match address is the proposed function start.
- **The seam that makes this read-only is the `MatchAction`, not the search.**
  `ghidra.app.analyzers.FunctionStartAnalyzer` **is** the production `PatternFactory`, and
  instantiating it is not read-only: its `applyActionToSet` calls `func.setNoReturn(true)` and
  writes an `AddressSetPropertyMap`. Both `PatternFactory` (`getMatchActionByName`,
  `getPostRuleByName`) and `MatchAction` (`apply`, `restoreXml`) are plain interfaces, so
  `@JImplements` works; return a fresh recording action per tag so each pattern keeps its own
  attributes, and mirror `DummyMatchAction` by consuming the element in `restoreXml`.
- **The XML attributes are post-match PREREQUISITES, checked long after the bytes match**, in
  `FunctionStartAnalyzer.checkPreRequisites`. Count hits without reading them and you will
  report padding as candidates. `validcode="function"` requires an **existing** function at the
  address; `validcode="N"` runs `PseudoDisassembler.checkValidSubroutine`; `after=` runs
  `checkAfterName`, whose branches are `func` / `inst` / `data` / `ptr` / `def`, and whose
  fallback `pureDataReferencesOnly` **opens by refusing an address with no references at all**.
  Also `label=`, `noreturn=`, `thunk=`, `contiguous=`, `section=`.
- **`funcstart` and `possiblefuncstart` are different tiers and both create functions.** The
  first goes to `funcResult` → `analysisManager.createFunction`; the second to
  `potentialFuncResult` → a scheduled `PossibleDelayedFunctionCreator`. And
  `applyActionToSet` silently drops any address failing
  `addr % language.getInstructionAlignment() != 0` (1 on x86, so never).

**Calibrate a borrowed engine like any other witness**: run it over the whole program and
require a large share of matches to land on functions that already exist. Measured on one
binary, 140 final patterns → 2,905 matches, **1,205 (41.5%) on known function starts**. Without
that arm, "0 matches in the region I care about" is a fact about your wiring.

## Read-only witnesses — analysis without mutation

Three shipped helpers answer questions worth asking without writing to the program. All three
have an arm that *does* write, and in two cases it is not the arm you would guess, so the rule
is the same each time: **read the source for which arm mutates, then bracket the run with a
datatype/function-count diff that raises.** The source read is the hypothesis; the bracket is
the evidence.

### `PseudoDisassembler` (`ghidra.app.util`)

Genuinely read-only by its own javadoc — *"no references, symbols are created or will be
saved"*, and it needs no open transaction.

```python
pd = PseudoDisassembler(program)
pd.isValidSubroutine(addr)                              # returns, no bad instrs, one entry, no overlap
pd.isValidSubroutine(addr, allowExistingCode)           # allow flowing into existing instructions
pd.isValidSubroutine(addr, allowExistingCode, mustTerminate)
pd.checkValidSubroutine(addr, ctx, allowExisting, mustTerminate, contiguous)
pd.disassemble(addr)                                    # -> PseudoInstruction (also (addr, bytes[]))
pd.isValidCode(addr[, ctx]); pd.followSubFlows(entry, maxInstr, processor)
pd.setMaxInstructions(n); pd.getLastCheckValidInstructionCount()
```

**Use `allowExistingCode=True` when testing a KNOWN function start** — the default form requires
not overlapping existing instructions, so it refuses every function you already have and the
recall arm reads as a broken instrument.

**Measured calibration, and it is the reason to distrust it as a discriminator: 400 of 400
sampled known function starts accepted (100% recall) and 288 of 400 sampled function INTERIORS
also accepted — a 72% false-accept rate.** So "N undisassembled ranges decode cleanly" is close
to information-free. It is a sanity filter, not evidence. Grade it in both directions before
believing any count it produces.

### `FillOutStructureHelper` (`ghidra.app.decompiler.util`)

The same witness as a hand-rolled `this`-offset tracker, but over decompiler SSA — so it crosses
basic blocks via `MULTIEQUAL` phis and, given a `DecompInterface`, recurses into CALLs.

```python
h   = FillOutStructureHelper(program, monitor)
lib = h.setUpDecompiler(DecompileOptions())             # pins setSimplificationStyle("decompile")
var = h.computeHighVariable(storageAddr, func, lib)     # HighVariable at that storage, or None
h.processStructure(var, func, True, False, lib)         # (createNewStructure, createClassIfNeeded)
h.getStorePcodeOps(); h.getLoadPcodeOps()               # List<OffsetPcodeOpPair>
h.getComponentMap()                                     # NoisyStructureBuilder
```

**Which arm mutates — the opposite of the intuitive reading:**

| arguments | effect |
|---|---|
| `createNewStructure=True, createClassIfNeeded=False` | **read-only.** `createUniqueStructure` builds a `StructureDataType` and **never adds it to the DTM**; `populateStructure` edits that detached object. Take the op lists, discard the Structure. |
| `createNewStructure=False` | **rewrites an existing type.** `getStructureForExtending` fetches the struct *out of the DTM* and then `growStructure` / `replaceAtOffset` run on it — i.e. it overwrites the very layout you meant to cross-check. |
| `createClassIfNeeded=True` on a `this` param | **mutates and launders.** `createUniqueClassNamespaceAndStructure` calls `symbolTable.createClass(..., SourceType.USER_DEFINED)`, moves the function with `RenameLabelCmd(..., USER_DEFINED)`, and `VariableUtilities.findOrCreateClassStruct`. |

Three more caveats, all measured:

- **The returned ops are not confined to the function you passed** — `pushIntoCalls` walks into
  callees. Ghidra's own caller filters them (`RecoveredClassHelper.removePcodeOpsNotInFunction`).
  Filter with `func.getBody().contains(op.getPcodeOp().getSeqnum().getTarget())`, and report
  both counts: the unconfined set is what compares to a construction-chain walk, the confined
  set to a single body.
- **It is decompiler-derived, so it inherits every signature and type you applied.** Agreement
  on a class you have already typed may be a tautology; agreement on an untyped one is a real
  second witness. Split the population before quoting an agreement count.
- **Its precondition is a locatable receiver, and that is where reach dies.** It needs a
  `HighVariable` at the receiver's storage. On a `__thiscall`/`__fastcall` body that is ECX; on
  a `__cdecl` one ECX is not an input at all and `computeHighVariable` returns `None`. Measured:
  198 of 198 classes had a constructor to point it at — and 173 carried an analyzer-committed
  `__cdecl` prototype, so effective reach was **25**. Census `getCallingConventionName()` over
  the population before pricing anything on it. Do **not** reach for `getParameter(0)`'s storage
  instead: that is `this` only under `__thiscall`, and under `__cdecl` it is the first *stack*
  argument, which yields a `HighVariable` every time and produces no cells — a failure that
  reads as a quiet witness rather than a wrong question.

### `AlignedStructureInspector` (`ghidra.program.model.data`)

`packComponents(structureInternal)` → `StructurePackResult` (`.structureLength`, `.alignment`)
computes an MSVC-packed size **without touching the DTM**. Turns a candidate layout into a
hypothesis test: alignment alone narrows a size interval, and a `double` member narrows it
further. Narrowing, never deciding. *(doc — not executed here.)*

## Correlation across builds

`ghidra.program.model.correlate`: `HashedFunctionAddressCorrelation`,
`HashCalculator`, `MnemonicHashCalculator`, `AllBytesHashCalculator`, `HashStore`,
`DisambiguateByParent`/`ByChild`/`ByBytes`.

`ghidra.features.bsim.*` — BSim similarity database and query API (`query.facade`,
`query.protocol`, `query.client`). For matching a stripped build against a build with
symbols, or identifying library code. Tutorials in `docs/GhidraClass/BSim/`.

## Emulation

**Current:** `ghidra.pcode.emu` — `PcodeEmulator`, `PcodeMachine`, `PcodeThread`,
`EmulatorUtilities`, `PcodeEmulationCallbacks`. `PcodeMachine.inject(Address, sleigh)`
stubs out imports the emulator cannot execute, which is what makes an era-typical Win32
binary emulable at all. Worked examples ship in
`Ghidra/Features/SystemEmulation/ghidra_scripts/` (`StandAloneEmuExampleScript.java`,
`EmuDeskCheckScript.java`).

**Deprecated:** `ghidra.app.emulator` (`EmulatorHelper`, `Emulator`, `DefaultEmulator`,
`EmulatorConfiguration`, `MemoryAccessFilter`, `FilteredMemoryState`). 12.1.2's own stub for
`EmulatorHelper` says *"This is part of the older p-code emulation system… deprecated::
Please use `PcodeEmulator` instead."* Keep it only for `enableMemoryWriteTracking()` /
`getTrackedMemoryWriteSet()`, which have no direct replacement.

Run a routine without the target running — decompression, checksum, decryption, table
generation. Set registers and memory, run to an address, read results. Course material:
`docs/GhidraClass/Debugger/B2-Emulation`.

## Version control and checkpointing

```python
df = currentProgram.getDomainFile()
df.isVersioned(); df.isCheckedOut(); df.getLatestVersion()
currentProgram.isChanged()
end(True)                                     # close the script's transaction FIRST
currentProgram.save(comment, monitor)
df.checkin(checkinHandler, monitor)           # handler via @JImplements(CheckinHandler)
df.addToVersionControl(comment, keepCheckedOut, monitor)
df.packFile(file, monitor)                    # .gzf snapshot of the PERSISTED state
```

`packFile` serializes the last-**saved** state, so snapshot **after** save or the archive
silently captures the previous version. `"File has unsaved changes"` from `checkin` is
literal.

## Debug information

**`ghidra.app.util.bin.format.pdb2.pdbreader` is the CURRENT reader** — a pure-Java,
cross-platform implementation (~600 files) driven by the **`PdbUniversalAnalyzer`**, so it
works off-Windows and needs no DIA SDK. The older `ghidra.app.util.bin.format.pdb`,
`docs/README_PDB.html` and `support/README_createPdbXmlFiles.txt` describe the **legacy**
native `pdb.exe`/XML route, whose README still predicts the pure-Java implementation that
has since arrived. Prefer `pdb2`; reach for the legacy path only if Universal fails.
`ghidra.app.util.bin.format.dwarf` is a large, full DWARF implementation with
`dwarf.line` (line numbers), `dwarf.external` and `sectionprovider`; Ghidra 12.1 added
**debuginfod** downloads and a `$HOME/.cache/debuginfod_client` search. Golang and Swift
metadata have their own packages (`format.golang.rtti`, `format.swift.types`), as does
Objective-C (`format.objc`).

**Check for debug info before doing anything else on a Windows or ELF target.** It
subsumes most of the evidence-gathering this methodology otherwise does by hand.

## Exporting

**For a C header of recovered TYPES, use `ghidra.program.model.data.DataTypeWriter(dtm,
writer)`** — public, supported, and documented as *"A class used to convert data types into
ANSI-C. The ANSI-C code should compile on most platforms."*

**Do not try to use `CppExporter.CPPResult`.** It is a **private nested record**
(`record CPPResult(Address, String headerCode, String bodyCode, List<String> globals)`),
not reachable from a script and absent from the javadoc; `CppExporter` itself **decompiles
the whole program** as a side effect and is driven by options `CREATE_C_FILE`,
`CREATE_HEADER_FILE`, `USE_CPP_STYLE_COMMENTS`, `EMIT_TYPE_DEFINITONS` (Ghidra's typo),
`EMIT_REFERENCED_GLOBALS`. Its type emission delegates to `DataTypeWriter` anyway.

**No built-in emits compile-time offset assertions**, so wrap the generated types with
`_Static_assert(offsetof(T,f)==K)` and `sizeof` checks yourself and compile them — that
compile is the only step that recomputes offsets from the C object model rather than from
your own arithmetic.

Other exporters in `ghidra.app.util.exporter` cover XML, HTML, ASCII and binary; the SARIF
exporter lives in `Ghidra/Features/Sarif`.

## Headless automation

`support/analyzeHeadless` — batch runs with no GUI. Verified options include:
`-import`, `-process`, `-recursive`, `-preScript <name> [args]`,
`-postScript <name> [args]`, `-scriptPath`, `-propertiesPath`, `-scriptlog`, `-log`,
`-noanalysis`, `-analysisTimeoutPerFile <s>`, `-processor <languageID>`,
`-cspec <compilerSpecID>`, `-loader`, `-librarySearchPaths`, `-max-cpu <n>`,
`-readOnly`, `-deleteProject`, `-okToDelete`, `-commit`, `-connect`, `-keystore`, `-p`.

Two matter for methodology:

- **`-readOnly`** guarantees only that changes are **not persisted**: the README says
  imported files "will NOT be saved" and that in `-process` mode any changes "are
  **discarded**". It does **not** prevent mutation during the run, so a script can still
  apply names and a later `-postScript` in the same chain can still harvest them. It is a
  containment mechanism, not an anti-circularity one — do not treat it as a guarantee that
  a pass was evidence-only.
- **`-process`** operates on a program already in a project, so a headless pipeline can
  drive the same database an interactive session uses, with `-preScript`/`-postScript`
  bracketing.

## PyGhidra / JPype interop — the errors that cost time

- **A Java collection parameter needs a Java collection.** `dtm.findDataTypes(name, [])` fails
  with *"No matching overloads found"* against a signature it obviously matches; pass
  `java.util.ArrayList`. Same shape as the `java.lang.Object` consumer required by
  `getReadOnlyDomainObject`.
- **Implement Ghidra interfaces with `@JImplements` / `@JOverride`** (`from jpype import ...`).
  Verified on `PatternFactory`, `MatchAction` and `ContextEvaluator` — all plain interfaces.
  This is what lets you substitute one step of a shipped pipeline while keeping the rest real.
- **`getBytes(addr, n)` returns a Java `byte[]`; `bytes(ba)` converts it** via the buffer
  protocol, giving unsigned values directly. Do this once per memory block and use
  `bytes.find` — a per-byte `mem.getByte()` loop over a megabyte is minutes, this is instant.
  Assert the returned length equals what you asked for: a partial read silently shifts every
  offset you compute afterwards.
- **Type-check against INTERFACES, not implementation classes** — a datatype read back from the
  DTM is a `FunctionDefinitionDB`, not the `FunctionDefinitionDataType` you constructed.
- **`state`, `currentProgram`, `currentAddress`, `monitor`, `script` are reserved
  `GhidraScript` globals.** Assigning to `state` raises `ClassCastException` from a property
  setter, reported far from the offending line.
- **Never derive a path from `__file__`** in an installed script; resolve from a constants
  module.

## Facts about the program you will want and may not think to ask for

```python
listing.getUndefinedRanges(mem.getExecuteSet(), True, monitor).getAddressRanges()
```
Every byte in executable memory the disassembler never touched — an `AddressSet`, so ask it for
`getAddressRanges()`. **Classify these by CONTENT before quoting the total**: measured, 5,033 of
5,321 ranges were pure `0xCC`/`0x00`/`0x90` alignment filler, so the honest population was 236
ranges, not 5,321.

```python
ins.getFlowType()      # .isTerminal() .isJump() .hasFallthrough() .isCall() .isConditional()
```
The authority on whether control leaves an instruction — **ask this, never a mnemonic list**.
Its use for grading a whole evidence base: if a defined function's last instruction still falls
through, that body is truncated in the database and every witness that walks function bodies has
been reading a partial function. Calibrate over the whole corpus first (measured: 5,628 of 5,629
functions end in a flow terminator) so the answer is a measurement, not a property of the probe.

```python
func.getCallingConventionName()   # '__cdecl' / '__thiscall' / '__fastcall' / None
func.getSignatureSource()         # DEFAULT | ANALYSIS | IMPORTED | USER_DEFINED | AI
dtm.getDataTypeCount(True)        # cheap invariant for a mutation bracket
```
Check the SOURCE before planning to overwrite a signature: `AI` and `ANALYSIS` are both priority
2, so an AI-tagged replacement of an analyzer-committed prototype is a **peer overwrite** the
next `analyzeChanges` is entitled to reverse.

```python
refMgr.getReferenceDestinationIterator(addressSetView, True)   # what is referenced in a region
refMgr.getReferencesTo(addr); refMgr.getReferenceCountTo(addr)
```
Remember these are claims about the **database**: a reference from bytes never disassembled does
not exist here. Byte-search for an address before recording any reference-count zero.
