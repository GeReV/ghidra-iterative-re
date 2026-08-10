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
| `docs/languages/` | Processor/Sleigh docs |

Search these with Python, not shell text tools, if your environment proxies `grep`.

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

**Count functions consistently.** `getFunctions(True)` excludes external/thunk-to-DLL
functions; `getFunctionCount()` includes them. They differ, so record which you used.

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
ClassUtils.getClassInternalsPath(composite)   # the "_internals" category convention
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

Adding a primary reference of the appropriate override type at a call/jump site converts
an indirect transfer into a direct one for the decompiler — the only way to devirtualize a
vtable call, since Ghidra cannot prove a vtable pointer is constant after assignment.
This writes *your inference into the call graph*: tag it and never harvest it back as fact.

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
`GenericMatchAction`, `AlignRule`, `PostRule`, `Match`.

Use for custom function-start signatures (attacking undefined-code backlogs) and library
fingerprinting. Also `ghidra.features.base.memsearch.*` for the memory-search API.

## Correlation across builds

`ghidra.program.model.correlate`: `HashedFunctionAddressCorrelation`,
`HashCalculator`, `MnemonicHashCalculator`, `AllBytesHashCalculator`, `HashStore`,
`DisambiguateByParent`/`ByChild`/`ByBytes`.

`ghidra.features.bsim.*` — BSim similarity database and query API (`query.facade`,
`query.protocol`, `query.client`). For matching a stripped build against a build with
symbols, or identifying library code. Tutorials in `docs/GhidraClass/BSim/`.

## Emulation

`ghidra.app.emulator`: `EmulatorHelper`, `Emulator`, `DefaultEmulator`,
`EmulatorConfiguration`, `MemoryAccessFilter`, `FilteredMemoryState`.

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

## Headless automation

`support/analyzeHeadless` — batch runs with no GUI. Verified options include:
`-import`, `-process`, `-recursive`, `-preScript <name> [args]`,
`-postScript <name> [args]`, `-scriptPath`, `-propertiesPath`, `-scriptlog`, `-log`,
`-noanalysis`, `-analysisTimeoutPerFile <s>`, `-processor <languageID>`,
`-cspec <compilerSpecID>`, `-loader`, `-librarySearchPaths`, `-max-cpu <n>`,
`-readOnly`, `-deleteProject`, `-okToDelete`, `-commit`, `-connect`, `-keystore`, `-p`.

Two matter for methodology:

- **`-readOnly`** makes a non-mutating round *enforced by the harness* rather than by
  discipline — the mechanical way to guarantee an evidence-only pass.
- **`-process`** operates on a program already in a project, so a headless pipeline can
  drive the same database an interactive session uses, with `-preScript`/`-postScript`
  bracketing.
