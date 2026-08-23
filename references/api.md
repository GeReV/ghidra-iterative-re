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
  198 of 198 classes had a recorded constructor site — and 173 of those sites are **factories the
  constructor was inlined into**, correctly `__cdecl`, so pointing it at the recorded site reached
  only **25**. Census `getCallingConventionName()` over the population before pricing anything on
  it, and check what each site actually IS before calling a convention wrong: an inlined-constructor
  factory is not a member function and has no receiver to declare.

  **But do not price the reach on one entry point per class.** Any non-static member takes `this`,
  so a class with no usable constructor is still reachable through a namespace member or a vtable
  slot target — which took the same population from 25 to **198 of 198**. If you use slot targets,
  drop the ones shared through inheritance first (a target in more than one attributed table is a
  base's method); measured 560 of 1322 shared, 0 classes dependent on a shared one. Do **not** reach for `getParameter(0)`'s storage
  instead: that is `this` only under `__thiscall`, and under `__cdecl` it is the first *stack*
  argument, which yields a `HighVariable` every time and produces no cells — a failure that
  reads as a quiet witness rather than a wrong question.

- **It follows the pointer's VALUE through `COPY` and `MULTIEQUAL`, so on a class hierarchy it
  will attribute a DERIVED class's fields to a BASE pointer — and the confinement filter does not
  help, because the contaminating accesses are in the same body.** Measured on a 1999 MSVC binary:
  driven from a base-class pointer whose type comes from the binary's own export mangling, it
  reached offset **624** — past the *entire* 496-byte allocation of a class deriving from that
  base, i.e. an impossible lower bound for the base. The mechanism is ordinary C++: the base
  pointer is copied into a list-walk iterator, the iterator is narrowed by a type-tag test, and
  only then are the derived fields touched. The helper sees one dataflow.

  **Two things follow.** First, **assert on the inverted interval**: a lower bound above a
  structurally independent upper bound is a defect, not a measurement, and printing it invites the
  reader to split the difference. Second, **find the mechanism that causes the narrowing and make
  that the discriminator** — here, *does this pointer's flow READ the type tag*. Partitioning on
  that separated the population completely and with no tuning (6 tag-reading variables reaching
  624, 18 non-tag-reading ones reaching 496, and the narrowed group landing inside the derived
  class's own size). **A threshold you have to tune to separate a population is a fitted
  parameter; one that separates it exactly is a mechanism.** Assert both the discriminator's
  identity and the coherence bound rather than assuming them.

- **`Function.getLocalVariables()` returns the COMMITTED (database) variables, not what the
  decompiler sees, and on a stripped binary those differ by an order of magnitude.** A population
  built from `getParameters()` + `getLocalVariables()` is a population of *what someone has
  already written down*. Measured: a probe built that way found 19 functions holding a pointer to
  the class of interest; the idiom that actually carries it is a decompiler-only local —
  `T *local_14; while (SearchT(this, &local_14, &end)) { ... }` — so every iterator site was
  structurally outside the population, including the one function that settled the question.
  **The omission produces no error, no warning and no skip counter**: the probe simply reports a
  smaller world, and its number reads as a fact about the program.

  To enumerate what the **decompiler** believes, go through
  `HighFunction.getLocalSymbolMap().getSymbols()` and take `sym.getHighVariable()` /
  `sym.getStorage()`. To enumerate what a **human** has recorded, use `getLocalVariables()` (or
  `getAllVariables()`, which adds the parameters). They are different questions and the second one
  is almost never the one you meant. **Audit what such a predicate FEEDS before asking how wrong
  it is**: one that selects a *control group* is worse than one that selects a population, because
  the same defect then shrinks the treatment group and poisons the baseline, and both errors push
  a measured payoff toward "no effect".

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

---

*The four sections below moved here from `SKILL.md` in the 2026-08-23
restructure: cascade triggers, the built-in-before-you-hand-roll table, driving
Ghidra in library mode, and the PyGhidra scripting rules.*

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
| Infer struct fields from access patterns | GUI: Auto Create / Auto Fill in Structure, **Auto Fill in Class Structure** for a known `this`. **Programmatic: `FillOutStructureHelper`** (`ghidra.app.decompiler.util`) — `processStructure(highVar, func, createNewStructure, createClassIfNeeded, decomplib)`, then **`getStorePcodeOps()` / `getLoadPcodeOps()`** for `(offset, PcodeOp)` pairs and `getComponentMap()` for a `NoisyStructureBuilder` |
| Compute an MSVC-packed size WITHOUT mutating the DTM | `AlignedStructureInspector.packComponents(structureInternal)` → `StructurePackResult` (`.structureLength`, `.alignment`) — hypothesis-testing for a candidate layout |
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
| Run a routine without the game | **`PcodeEmulator`** (`ghidra.pcode.emu`; `PcodeMachine.inject(addr, sleigh)` stubs unemulatable imports). `EmulatorHelper` is **deprecated** as of 12.1 — keep it only for `enableMemoryWriteTracking()`/`getTrackedMemoryWriteSet()`. Examples in `Features/SystemEmulation/ghidra_scripts/` |
| Discard changes after a headless pass | `analyzeHeadless -readOnly` — it guarantees nothing is **persisted**; it does *not* prevent mutation within the run, so a later `-postScript` can still read this run's own applies. Containment, not anti-circularity |
| Batch/pipeline runs | `analyzeHeadless -process -preScript -postScript -noanalysis` |
| Preview bytes without committing | Disassembled View window |
| **Emit recovered types as C** | `DataTypeWriter(dtm, writer)` — public and supported. Not `CppExporter.CPPResult`, a private record whose exporter decompiles the whole program |
| **Measure what a round actually changed** | `ProgramDiff` + `ProgramDiffFilter` → `AddressSetView`; `ProgramMerge` to restore one category from a checkpoint |
| Recover C++ classes wholesale | `RecoverClassesFromRTTIScript` + the `classrecovery` package — **gates on RTTI**, so a `/GR-` binary gets nothing, and that is what licenses hand-rolling |
| C++ classes on 32-bit MSVC with **no** RTTI | Pharos **OOAnalyzer** (`--ignore-rtti`) via **CERT Kaiju**. Candidates only: destructor F1 0.41, member offsets unpublished |
| Devirtualize a call site | `AddVfunctionCallRefScript` (12.1) — needs components typed `Pointer`→`FunctionDefinition` (NOT the `ClassUtils` default pointer), is cursor-driven, writes `ANALYSIS`, and its unique-instance inference is unsound for any class with descendants. Read the caveats before planning on it |
| Dataflow across basic blocks | `DecompilerUtils.getBackwardSliceToPCodeOps` (SSA p-code); `SymbolicPropogator` for `this + K` without the decompiler |
| Identify the build toolchain | `ghidra.app.util.bin.format.pe.rich` + `PortableExecutableRichPrintScript` — the PE Rich header names the compilers and linkers used |
| **Test if a subroutine starts here, WITHOUT mutating** | `PseudoDisassembler` (`ghidra.app.util`) — `isValidSubroutine(addr[, allowExistingCode[, mustTerminate]])`, `isValidCode`, `disassemble(addr)` → `PseudoInstruction`, `followSubFlows`. Creates no references or symbols and needs no transaction. **Calibrate it: measured 100% recall on known starts and a 72% FALSE-ACCEPT rate on known function interiors** — it is a sanity filter, not a discriminator |
| Attack undefined code | `Processors/x86/data/patterns/x86win_patterns.xml` (real MSVC prologue/filler pairs — **extend the XML**, don't write pattern code); `FindUndefinedFunctionsScript`, `MakeFunctionsScript`, `CreateFunctionAfterTerminals`, `DumpFunctionPatternInfoScript` |
| Build a FID database from period libs | `ImportMSLibs`, `CreateEmptyFidDatabase`, `CreateMultipleLibraries`, `RepackFid`; procedure in `data/building_fid.txt` |
| **Carry provenance inside the program** | `FunctionTag` (name + comment, DB-backed, and `CppExporter` can filter on it) — tag every function a round touches with the round id, and the ledger becomes reconstructible *from the program* instead of only from your CSV. `BookmarkType.{NOTE,ANALYSIS,WARNING}` for "recorded unresolved" caveats that travel with the address | **Also: a PLATE comment at every address you named, so the GUI reader sees it** — measured, 3618 of them over 1909 functions and 1709 data symbols, with `functions_total` and the AI-symbol count asserted unmoved and a full read-only-sweep harness confirming 0 artifacts perturbed. Make the text rename-proof: say *the name here is ours*, never quote the name, or you have planted one stale copy per address
| Record the analyzer configuration | `pyghidra.analysis_properties(program)` — results are a function of it, so a metrics row omitting it is not a reproducible snapshot |
| Machine-readable export for replay debt | `ghidra.app.util.xml.*Mgr`, and the shipped **SARIF** exporter (`Features/Sarif`) — both carry the `SourceType` laundering hazard in `references/trust-and-circularity.md` |

## How to drive Ghidra — prefer library mode

**Default to PyGhidra as a library, from plain CPython.** `pyghidra.start()` →
`open_project()` → `program_context()` → `with pyghidra.transaction(program, "round 5
apply"):` → `analyze()`. Also there: `consume_program()`, `analysis_properties()`,
`ghidra_script()`, `walk_programs()`. (PyGhidra 3.0, shipped with 12.0, deprecates the older
`open_program()`/`run_script()` in favour of these.)

This is not a style preference. **Library mode structurally eliminates three of the four
hazards below**: no install step means no stale-copy shadowing, no async task means no
unread result, and your own repo is on `sys.path` so paths resolve where you wrote them.
You also get real tracebacks. The one constraint: headless cannot open a project that is
already open in the GUI, so pick one at a time.

Reserve **MCP for interactive reads** against a live GUI session. When you *do* run scripts
through Ghidra, these apply:

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
- **Type-check against Ghidra's INTERFACES, not its implementation classes.** A datatype
  read back from the `DataTypeManager` is a DB-backed implementation —
  `dtm.addDataType()` hands you a `FunctionDefinitionDB`, which implements
  `FunctionDefinition` but is **not** an instance of the concrete
  `FunctionDefinitionDataType` you constructed. Measured: an end-state check written
  against the impl class reported **0 of 1053** on a retype that had completely succeeded.
  Ghidra's own shipped scripts check the interface (`instanceof FunctionDefinition`) —
  follow them. Same family as the rule below: the object you get back is not the object
  you put in.
- **Compare against the API's own returned value, never a hand-written literal.** Ghidra
  names `UnsignedShortDataType` **`ushort`**, not `uint16_t`; a literal in an end-state
  check aborted a *correct* apply after its write had committed, destroying that run's
  before/after measurement. Derive the expected string from the same object you write
  (`dt.getName()`), and treat a helper's return vocabulary as API too — one classifier
  returns `x87_float`/`x87_int` where `float`/`int` were assumed, and the mismatch was
  silent.
- **`runScript(name, args)` SWALLOWS the called script's exception.** Ghidra prints
  the traceback and `runScript` returns normally, so a driver that runs sweeps in a
  loop reports every one of them as passing. Measured: a stability harness reported
  `raised: 0` while two of its nineteen sweeps had raised calibration failures, with
  both tracebacks visible in the same log. If you need the failure as a *value*, run
  the script yourself so exceptions propagate —
  `exec(compile(open(path).read(), path, "exec"), g)` with `g` seeded from the
  GhidraScript globals (`currentProgram`, `currentAddress`, `monitor`, `state`) plus
  a per-script `getScriptArgs`. A driver that cannot see the failure it exists to
  detect is this document's unfireable-check rule wearing the harness's clothes.
- **`state` is a reserved `GhidraScript` global** (the `GhidraState`). Assigning a string
  to it raises `ClassCastException` from PyGhidra's property setter, reported at a line
  far from the one that looks wrong. Others in the same namespace: `currentProgram`,
  `currentAddress`, `monitor`, `script`.
- **Never derive a path from `__file__` in a script you install.** The installed copy
  lives in Ghidra's user script directory, where `dirname(__file__)/..` resolves outside
  your repo entirely — measured: a header generator read `C:\Users\<you>\symbols`, so
  its compile check (the only step recomputing offsets from a C compiler rather than from
  the project's own arithmetic) had *never once run*. Resolve paths from a shared
  constants module.
