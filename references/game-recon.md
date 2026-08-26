# Game reconnaissance — free names, library code, and what games look like

*Reference for the `ghidra-iterative-re` skill. Only `SKILL.md` is loaded when the
skill is invoked; this file is read on demand when its trigger fires. New lessons of
this kind belong here, not in `SKILL.md`.*

**Read this when:** you are opening a new game binary, hunting for names you do not have
to infer, separating library code from game code, or working out what era and engine you
are looking at.

**In this file:**

- Sweep for free names before analyzing anything
- Separating library code from game code
- External corroboration is legitimate evidence — cite it as such
- Data shapes characteristic of games
- Modern engines — check the engine before you open the disassembler
- Platform breadth
- Runtime observation — the primitive games give you for free

## Sweep for free names before analyzing anything

Ordered by payoff. Do these before you interpret a single function — every name found
here is `IMPORTED`-grade, and analysis done without them is wasted effort.

1. **Debug information, if any survives — do this FIRST, because it dominates everything
   below it.** A shipped or leaked `.pdb` (Windows), DWARF, `.debug` sections, or a `.map`
   file gives you names *and* full types *and* often line numbers outright, collapsing most
   of this list into a single import. Ghidra's current reader is
   `ghidra.app.util.bin.format.**pdb2**.pdbreader` driven by the **PDB Universal** analyzer
   — pure Java and cross-platform, so it works off-Windows; the older
   `ghidra.app.util.bin.format.pdb` + `docs/README_PDB.html` describe the *legacy* native
   `pdb.exe`/DIA-SDK/XML route and its Windows-only prerequisite. DWARF has a full loader
   plus **debuginfod** download support as of 12.1. Rarely present in retail game builds —
   but the payoff is so much larger than every other source that checking costs nothing by
   comparison. Look for a `.pdb` path string, `RSDS`/`NB10` signatures, and a CodeView debug
   directory entry. Also check whether the *modding community* has one; leaked symbol files
   circulate for popular games. **Period PC games also shipped their own symbol files by
   accident with some regularity** — Ghidra has loaders for three: `MapLoader` (Microsoft
   `.MAP`: names, `segment:offset`, public *and static* symbols), `DbgLoader` (CodeView
   `.DBG`), `DefLoader` (`.DEF`, which names ordinal-only DLL exports for free). Look in the
   install directory, not just the executable.
2. **Exported and mangled symbols.** Mangled C++ names carry class, member, virtualness,
   and full signature. Even a stripped retail build often exports more than expected.
   **Data exports name globals AND class statics, with static types** — and the mangling
   says which: `@@3` is a true global (`?pRendEng@@3PAVCRendEng@@A` == `CRendEng *pRendEng`),
   while `@@0`/`@@1`/`@@2` are private/protected/public **static class members**. Attributing
   a `@@2` symbol to a global is a silent ownership error of exactly the family this skill
   warns about for parentage and destructor strides. Measured on one binary: **50 class
   statics against 17 true globals**, so the majority case was the one the naive reading gets
   wrong. Parse the PE export directory for non-code targets, don't
   stop at functions. And when an exe exports symbols at all, ask WHO imports them:
   a companion DLL importing back from the exe (a plugin renderer, say) is itself a
   symbol source, and its import thunks explain writes to exe globals that no exe-side
   store accounts for. The stronger version, measured here: a DLL that imports a
   registry object and its insert/hash helpers is REGISTERING into an exe-side runtime
   registry — so when a name-keyed registry has members no exe string explains, sweep
   the companion binaries' strings against the ids before concluding the names are
   lost (13 of 15 mystery handler ids in one registry were class names present only
   in the plugin DLL). Corollary, measured the hard way: **before hypothesizing about a
   global's identity, query the symbol table AT its address** — a reconnaissance here
   built a loader-record theory for a pointer whose `IMPORTED`-grade label had been
   sitting on the address since import. The symbol lookup is the cheapest evidence
   there is; references-first analysis skips it silently.
4. **Other builds of the same game.** Demos, betas, console/other-platform ports,
   patched versions, and debug builds frequently ship symbols the retail build stripped.
   Diffing builds also isolates what a patch changed. Ghidra's **Version Tracking** and
   **BSim** exist for exactly this correlation.
5. **Embedded scripting VMs.** Lua and custom interpreters register native functions by
   **name** in a table — a registration table is a free symbol table mapping strings to
   function pointers. Find the registrar, get hundreds of names at once.
6. **Config / INI / CVar / console-command parsers.** A string→offset or string→setter
   table names struct fields and globals directly, in the developers' own vocabulary.
7. **Library and engine fingerprints.** Version strings, banners, and known code
   (Miles Sound System, Bink, RenderWare, Glide, DirectX, Havok, zlib, id/Build/Unreal
   lineage). Identifying an engine or middleware imports an entire published API surface
   at once. Ghidra's **Function ID (FID)** does byte-level library matching — but its
   results are **lower bounds**, silent on code that neither calls marker imports nor
   matches the corpus.
8. **Localization tables, asset and archive filenames, resource IDs.** These name game
   *concepts*, which is what you want for struct and enum naming.
9. **RTTI and vtable symbols**, where the language and build produced them
   (MSVC `??_7Class@@6B@` vftable symbols, `.?AV...@@` type descriptors, Itanium-ABI
   `_ZTV`). Absence is a measured finding — a build with `/GR-` has none, and that closes
   the cheapest naming route rather than meaning "look harder." **A zero here says more
   than "no RTTI":** MSVC emits `_TypeDescriptor` records (`.?AV…`) for types named in
   `catch` clauses and `throw` *independently of `/GR`*, so no `.?AV` at all means neither
   polymorphic RTTI **nor typed C++ exception handling**. Conversely, `.?AV` strings in a
   `/GR-` build are EH data, and the structures around them are `FuncInfo`, not `??_R0`.
10. **Assert / debug strings containing source paths.** `__FILE__` in an assert macro
   embeds the **original source file path**, often with line numbers. Where it survives
   this reconstructs the developer's module decomposition — directory names are
   subsystems, filenames are translation units — and a function containing
   `"c:\\proj\\ai\\pathfind.cpp"` belongs to the pathfinding module whatever it is called.
   Record hits as **source map entries** (see `references/api.md`), not comments.
   *Calibration: yield varies enormously with build settings. Sweeping a 1999-era retail
   game binary here returned exactly one `.cpp` path and one `.h` name — enough to
   establish the source root and one subdirectory, not enough to reconstruct modules,
   because release builds compile most asserts out. Cheap to check, so always check; do
   not budget a phase around it before measuring.*
11. **Linker-contract tables: static initializers, TLS callbacks, and exception data.**
   Not names, but **ground truth about function starts** — and for EH, about types — with a
   linker contract behind them rather than a heuristic. Measured on one binary: the
   `.CRT$XC*` table yielded **1429 function starts** in a single round.
   - `.CRT$XCA..XCZ` (C++ initializers) and `.CRT$XIA..XIZ` (C) are null-bounded arrays of
     `.text` pointers consumed by `_initterm`; `.CRT$XPA`/`XTA` are the atexit/terminator
     equivalents. **Every entry is a function start, by contract.** PE TLS-directory
     callbacks are the same shape.
   - MSVC x86 C++ EH: `__CxxFrameHandler`'s `FuncInfo` carries an **unwind map naming a
     destructor per stack object with its frame offset**, and a try/catch map naming caught
     **types** — destructor identification and type evidence in one structure, present even
     under `/GR-`.
   - `/SAFESEH` binaries carry a validated handler table in the load-config directory:
     another free list of real function starts.
   - On x64, `.pdata`/`.xdata` bound *every* function exactly — reading them largely
     dissolves the undefined-code problem that dominates x86 work.

**Use the developers' vocabulary.** Terminology from the game's own UI, manual, and
strings is what the authors called these things. Name recovered entities to match it, not
to match your model of the game.

**Apply a qualified name as a NAMESPACE, not as a string containing `::`.** Ghidra's
demangler creates a `GhidraClass`/namespace and stores a *short* name — which is why
`getName(True)` reconstructs `CGobject::Sleep` while `getName()` returns `Sleep`. A flat
symbol literally named `"CGobject::Sleep"` therefore lives in a different space from every
demangled symbol and will never join with them: it is this skill's own "raw mangled names
and demangled short names are different string spaces" trap, self-inflicted. Use
`symbolTable.createClass(...)` + `func.setParentNamespace(cls)` + the short name, tagged
`SourceType.AI`.

**Decide what to reverse next by working backwards from the target, not outwards from what
is legible.** Rank populations by what a reimplementation cannot be written without: the
main loop and its tick order, the simulation state that loop mutates, the serialization
boundaries (save, replay, network), then rendering and UI. A class that is easy to lay out
and never appears in the loop is a completed round with nothing delivered. State the round's
target in the metrics row next to the counts, so "what did this buy" is answerable.

For lockstep-deterministic genres (RTS especially) the format gives you two free oracles:
the **order/command struct and the sync checksum are both layout ground truth**, and a
desync against the original binary is a differential test you can run.

## Separating library code from game code

A statically-linked game binary mixes engine, CRT, middleware, and game code with no
section boundary between them. Partitioning them early is what stops later phases from
carefully reverse-engineering `strlen`. Two mechanisms, proven in this project, and they
are **independent** — run both:

1. **Function ID (FID)** — byte-hash matching against Ghidra's library corpus, run as a
   **descending threshold ladder** rather than at one cutoff. Its help documents exactly two
   outcomes, **Single Match** and **Multiple Matches**, recorded in the comment and bookmark;
   **only a Single Match is safe to act on.** Applying FID names can leave orphans that need
   a repair pass. Note that FID's separate *"Function ID Conflict"* marking is **not** a
   third match tier — it means a proposed name collided with an existing symbol at apply
   time, which is a naming problem, not evidence about match quality.
2. **Marker-import transitive closure** — seed from imports only a given library calls,
   then walk the call graph. Cheap and independent of any corpus.

Caveats that cost real time here, all of which generalize:

- **Both mechanisms share one blind spot**: library code that neither calls a marker
  import nor byte-matches the FID corpus is invisible to both. Every count is a **LOWER
  BOUND**, never a population estimate. "Unclassified" does not mean "game."

  **The corollary that is easy to miss: library identification is not a phase you
  complete, it is a continuing byproduct of game-side work.** Measured on one project long
  after its partition was "done": eight CRT stdio functions — `fopen`, `fclose`, `fwrite`,
  `fread`, `ftell`, `fseek`, `sprintf`, `_strerror` — all sitting in the CRT address
  region, one of them *between* two functions FID had already named, and every one still
  carrying no classification at all. They were found by following a **game-side data
  format downward into the library**: `fwrite` surfaced because 224 serialisation methods
  called it, not because anything was scanning for library code. The leaf functions at the
  bottom of a game-side call chain are exactly what both mechanisms miss — small, calling
  no imports, and an inlined or variant build will not byte-match a corpus. So when a
  game-side population is opaque, **look at what all of it CALLS and identify the callee
  first**: the inverse of the usual top-down direction, and where the residue of the
  partition lives.

- **Sweep the BOUNDARY DECLARATIONS before hunting the leaves — they are earlier, freer
  and more specific.** Exported and imported signatures name the I/O layer by TYPE, with
  no analysis at all. Measured on one project: its very first symbol export, day one,
  2640 functions, already contained
  `void Save(CGobject *this, _iobuf *param_1, int param_2)` and **38 signatures
  mentioning `_iobuf`** — the C stdio `FILE`. The serialisation boundary was fully
  declared from the first hour and went unread for **246 commits and eight days**, until
  it was rediscovered from the other end. One query over the signature column
  (`FILE`/`_iobuf`, `SOCKET`, `HANDLE`, `HKEY`) is minutes and tells you where the
  program touches the outside world; only then do you need the leaves, and by then you
  have call sites to pin them by.

- **Prefer leaves whose ARGUMENTS CARRY STRUCTURAL METADATA — not merely high fan-out.**
  Three families repay identification far beyond the rest, because their arguments *are*
  layout facts:
  - **allocators** — the size argument is an object size (an entire size-recovery
    machinery can rest on `operator new` call sites; an external tool's recall collapsed
    on this binary purely because it could not find them);
  - **block copy** — `memcpy` / `rep movsd` extents are member-array bounds;
  - **I/O** — sizes *and order*, which is what turns a family of serialisation methods
    into a field-by-field layout witness.

  `strlen` has enormous fan-out and is trivially identifiable, and tells you nothing
  structural. Fan-out decides *when* to apply a name; this decides *which* leaves to hunt.

- **THE ALLOCATOR YOU FOUND IS PROBABLY A WRAPPER, AND THE POPULATION QUESTION BELONGS TO ITS
  WORKER.** A game with more than one memory pool typically gives each pool a thin per-pool
  door — a dozen instructions that push a pool descriptor and a size to one shared worker and
  report a pool-specific message on failure. Name the door and you have one allocator; ask
  instead for **the set of distinct descriptor arguments reaching the worker** and you have
  the complete list of pools, with an end to it. Measured on one binary: a sibling failure
  string suggested "a second arena, on one witness", and enumerating the worker's call sites
  returned exactly **two** pools behind **four** doors — two of which were already named
  (`zmalloc`, `rendmalloc`, both PE export-table entries) and which no string search would
  ever have connected to the same pool. The finding that fell out was about the game, not the
  naming: three separate exported doors all draw on *the pool the game itself uses*.
  Generalise past allocators: **before hunting more instances of a pattern, find the
  chokepoint they all pass through and enumerate ITS inputs** — that converts an open-ended
  search into a census.
- **A BINARY-SUPPLIED NAME IS GROUND TRUTH; WHAT IT MEANS IS STILL YOUR INFERENCE — AND BOTH
  ARRIVE IN THE SAME STRING.** The rule above ("a library identification is still YOUR
  inference, and the famous name disguises that") is usually applied to names *you* attach.
  It fails harder in the other direction, where nothing prompts a second look: an exported
  symbol named `zfree` needs no defending as a *name* — the export directory supplied it — so
  reading the `z` as "zlib" and concluding "this belongs to the compression library, not to
  the game's arena" felt like description rather than inference, and stood unchallenged
  through two rounds and into a published write-up. Measured afterwards on the same binary:
  no `zlib`, `inflate`, `deflate`, `adler32`, `zcalloc` or `z_stream` string or symbol exists
  anywhere in it, and the only observed importers of `zfree` are the game's own two renderer
  plug-in DLLs. The prefix's meaning remains unknown. **Record "the binary supplied this
  string" and "this string means X" as separate claims at different tiers** — and note the
  tell: the moment a name's *spelling* is doing the work in an argument, an inference has
  entered wearing ground truth's clothes.
- **A CLAIM ABOUT AN ALGORITHM CANNOT BE MADE FROM ONE HALF OF THE PAIR THAT IMPLEMENTS IT.**
  In an allocate/release pair essentially all the policy lives in the allocator; release is
  frequently a single flag write. Measured: a release routine's eight instructions were read
  as "no size classes, no free list, no coalescing — a high-water mark", and used to justify a
  cautious name. The allocator walks a **circular free list**, **splits** blocks above a
  remainder threshold, and **coalesces** adjacent free neighbours while searching — a
  general-purpose allocator. The characterisation was wrong for eleven rounds and nothing ever
  contradicted it, **because the decision it justified was correct**. That is the shape worth
  fearing: a wrong reason sitting underneath a right answer is never disturbed by ordinary
  work. When you characterise a mechanism, name which bodies you read.
- **PRINT WHETHER A FUNCTION IS EXPORTED BESIDE EVERY FAN-IN, OR A ZERO READS AS DEAD CODE.**
  A fan-in of 0 is a claim about the database's view of *one module*. Measured: an allocator
  door with zero in-module callers was `isExternalEntryPoint` — called by the two renderer
  plug-in DLLs that import 46 of the 47 symbols the exe offers. Same family as "zero
  cross-references is a claim about the DATABASE, not the binary", and it bites hardest in
  exactly the binaries this skill tells you to look for companion DLLs in.

- **Identify by CALL SHAPE and argument constants where byte signatures fail.** A semantic
  fingerprint is checkable in a way a guess is not, and it is available exactly when FID is
  not: `fopen` pinned by a literal `"wb"` in argument 2, `fwrite`/`fread` by the
  `(ptr, size, count, FILE*)` signature with the return compared against `count`,
  `sprintf` by a format string. Record which constant did the pinning — that is the
  evidence, and it is what a later reader needs in order to disagree with you.

- **A library identification is still YOUR inference, and the famous name disguises that.**
  `fwrite` is not a guess about what `fwrite` does; it is a guess about what lives at
  *this address*. Tag it `SourceType.AI`, ledger it, keep it out of harvests — exactly
  like any other applied name. The pull to treat it as ground truth is strong precisely
  because the function is well known. Every count is a **LOWER
  BOUND**, never a population estimate. "Unclassified" does not mean "game."
- **A missing FID match is silent, but not unexplained.** There is no "almost matched"
  signal to inspect, so absence of a hit is not evidence of absence of library code — but
  before recording a structural zero, read `Ghidra/Features/FunctionID/data/building_fid.txt`.
  It documents four explicit suppression modes that account for most zeros: **auto-fail**
  ("a full-hash match will not be returned under any circumstances"), **force-relation** (only
  matches if a parent or child also matches), **force-specific**, and an instruction-count
  threshold that drops tiny functions.

  **BUT DO NOT TRUST THAT FILE ABOUT WHAT THE SHIPPED DATABASES CONTAIN — MEASURE THE ARTIFACT.**
  An earlier revision of this document read `building_fid.txt`'s library list and concluded the
  shipped corpus "covers further back than people assume, *including Visual Studio 1998*". That
  is **wrong on a 12.1.2 install**, and it is wrong for exactly the targets this document is
  aimed at. The file does list VS1998 Debug/Release — but read what the list is FOR: those
  libraries were used to generate `common_symbols_win32.txt`, the common-symbols file, not
  necessarily the shipped databases. Measured with `strings` over `vsOlder_x86.fidbf` (41 MB):

  | token | occurrences |
  |---|---|
  | `VC98` | **0** |
  | `VC42` | **6102** |
  | `Visual Studio 10.0` | 3751 |
  | `Visual Studio .NET 2003` | 3336 |

  `vsOlder_x64.fidbf` has **0** of both. The oldest 32-bit coverage shipped is **VC 4.2**; there
  is no VC5/VC6 content at all. For a late-1990s MSVC binary that reframes a poor FID yield:
  measured on one such target, 106 CRT identifications and **zero** third-party may be the
  CORRECT answer for a VC6 image matched against a VC4.2/2003+ corpus — a near-miss, not a
  misconfiguration, and not something a score-threshold tweak will rescue. **Check which
  compilers your install's databases actually contain before explaining a zero, and before
  budgeting a round on tuning.** The generalisation is one `references/assertions.md` already
  makes about artifacts versus prose ("when a new artifact replaces prose, diff it against the
  prose"): a shipped doc describing a build PROCESS is not a manifest of that build's OUTPUT.
- **You can BUILD a FID database from period libraries**, which converts FID from a lower
  bound into near-complete identification of whatever you fed it. Shipped machinery:
  `ImportMSLibs.java` ("Massive recursive import for a MS Visual Studio installation
  directory"), `MSLibBatchImportGenerator`/`Worker`, `CreateEmptyFidDatabase`,
  `CreateMultipleLibraries`, `RepackFid`, `FunctionIDHeadlessPre/Postscript`. If the
  target's middleware SDK libs can be found (Miles, Bink, RenderWare…), this is the highest-
  leverage library-ID move available and almost nobody runs it.
- **BSim is a THIRD mechanism, and it is independent of the other two.** Its features are
  decompiler-derived data-flow vectors that deliberately exclude constants, register names
  and data types — so it is blind in a *different* place from FID's byte hashes, which is
  exactly what the shared blind spot above calls for. `support/bsim` is the CLI,
  `CreateH2BSimDatabaseScript` builds a local database, and an **Overview Query** returns
  per-function match counts — a five-minute reach probe for "how much of this binary is
  library code". Best of all, the **Version Tracking BSim correlator computes signatures
  in-process at correlate time, so it needs no database at all**, and it publishes a
  calibrated error model: *for threshold N, the probability that a seed is incorrect is
  approximately 1/2^(N/5+9)*. **A witness that states its own false-positive rate is the
  rarest thing in this tool surface** — prefer it wherever it applies.
- **Transitive closure is where swallowing risk lives.** Closure from CRT seeds nearly
  absorbed named game code; guard it explicitly, cap it, and check whether the cap is
  structural. Do not reuse closure for a second library just because it worked once.
- **Derive a game-side exclusion set from the binary's own exported class names** and
  treat a match against it as an *absolute* exclusion — never reclassify a function as
  library because its name looks library-ish.
- **Report an `unknown_library` bucket rather than forcing every match into a known
  one.** An empty unknown bucket must be an honest result, not a closed-out one.
- **A sweep that renames only the functions it itself named leaves earlier-named ones
  behind**, so namespace-based counters silently undercount. Decide whether you are
  classifying *new* findings or *all* findings, and say which.

## External corroboration is legitimate evidence — cite it as such

Modding communities, file-format wikis, existing mod tools and editors, published SDKs,
strategy guides, and shipped source for the same engine are all real sources. They are
*not* binary ground truth: record them at their own provenance level, never as
`IMPORTED`. A format documented by a modder and confirmed against the bytes is strong; a
format documented by a modder and merely *assumed* is a hypothesis.

## Data shapes characteristic of games

- **Fixed-point arithmetic.** On pre-FPU and console targets, an integer multiply followed
  by a shift is fixed-point math — not a bug, not an integer computation. No float type will
  appear anywhere. Q-format mechanics: `references/platforms-eras.md`.
- **FPU era changes what an instruction tells you.** x87 runs on a register *stack*, so
  argument order needs tracking; SSE packs four lanes, so one instruction may touch four
  fields and instruction width is not field width. Detail: `references/platforms-eras.md`.
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

## Modern engines — check the engine before you open the disassembler

For a managed or scripted engine the correct first move is often **no binary work at all**.
Identify the engine from the directory layout before anything else; getting this wrong costs
weeks of reversing code that a dumper prints in a minute.

- **Unity / IL2CPP** (`GameAssembly.dll` + `global-metadata.dat`): the metadata carries
  *complete* type, method and field names with offsets. Use **`Il2CppInspectorRedux`**
  (actively maintained, metadata versions 29–110, up to Unity 6000.x) rather than the
  dormant original; `Il2CppDumper` emits ready-made `ghidra.py` / `ghidra_with_struct.py`.
  **Gotcha:** the generated `il2cpp.h` cannot be fed to Ghidra's parser as-is — it is C++,
  and `CParserUtils` is **C only**, so the type apply silently does nothing. Convert first.
- **Unreal**: GNames/GUObjectArray dumping is **runtime-only** — say so up front, because a
  static-analysis plan will otherwise chase it for days. Validate any dump with invariants
  in this document's idiom: `GNames[0] == "None"`, and the earliest objects must include
  `/Script/CoreUObject` plus entries named `Class`, `ScriptStruct`, `Function`.
- **Godot**: `gdsdecomp` recovers `.gd` source from `.gdc` bytecode. Again — do no binary
  work.
- **Packed or protected binaries**: identify the protector before assuming the code is
  obfuscated by hand. Era-correct signatures matter more than section names — SafeDisc's
  `"BoG_ *90.0&!! Yy>"` magic is more reliable than its `stxt371` section, and LaserLok
  ships the literal string `"Packed by SPEEnc V2 Asterios Parlamentas.PE"`. The negative
  lesson from that corpus is worth carrying generally: one project **disabled** its Denuvo
  `.srdata` detector for false positives — *a witness that fires on clean binaries is worse
  than no witness*, which is this document's vacuity rule wearing a different hat.

## Platform breadth

Console and pre-modern targets are common in game RE, and three hazards there invalidate
assumptions the rest of this skill makes. Per-architecture detail:
**`references/platforms-eras.md`**.

- **Get endianness right at import, before anything else.** The same processor family runs
  big-endian on some consoles and little-endian on others — PowerPC is BE on GameCube, Wii,
  Xbox 360 and PS3; **MIPS is LE on PS1, PS2 and PSP but BE on N64** (`.z64` is native
  order; `.v64`/`.n64` exist because of byteswapping); SH-2 on Saturn is BE, SH-4 on
  Dreamcast is LE. Picking the wrong
  language variant (`PowerPC:BE:32` vs the LE variant) silently garbles the entire
  disassembly, and no amount of later analysis recovers from it. It is the cheapest possible
  mistake to make and the most expensive to notice late.
  *This line said "MIPS is LE on … N64" for several revisions while
  `references/platforms-eras.md` correctly listed N64 as big-endian. **A reference that
  contradicts its own trigger file is worse than no reference**, because the trigger is
  what gets loaded and acted on — when you correct one, grep the other.*
- **Register context must be set, or whole regions disassemble as garbage.** ARM/Thumb mode
  selection, MIPS `$gp`-relative data addressing, PowerPC TOC. Set the value over the
  address range and re-disassemble; don't fight individual instructions.
- **An address is not an identity in an overlaid or bank-switched program.** PS1 and DOS
  games load different code to the same address; cartridge systems bank ROM into a window.
  Model each as a Ghidra **overlay memory block**, and note that this breaks
  address-keyed evidence — **every join key must become *(overlay space, address)***, and a
  whole-program function count stops being a population. Audit any evidence file for this
  before trusting it.
- **Mark memory-mapped hardware registers `VOLATILE`.** Otherwise the decompiler may fold
  repeated reads into one and silently delete the poll loop you were reading. Labelling one
  recognised VDP/GPU/DMA register also identifies the surrounding code instantly — often the
  fastest way into an unknown console binary.

## Runtime observation — the primitive games give you for free

You can *cause* a change: take damage, move, spend currency, open a menu. That converts
"what is this field" into a controlled experiment.

- **Known-value memory search** (the Cheat Engine pattern): find the address holding a
  value you can change, then xref it back to code. This finds structs static analysis
  cannot reach.
- **Breakpoint on a known event** to catch the `this` pointer of an object whose identity
  you already know, then dump its bytes across several instances.
- Ghidra's own **Debugger** connects to a live process; the **p-code emulator**
  (`ghidra.pcode.emu.PcodeEmulator` — not the deprecated `EmulatorHelper`) runs code
  *without* the game, which is ideal for decompression, checksum and decryption routines
  where you want the algorithm's output rather than a live session. `PcodeMachine.inject()`
  stubs out imports the emulator cannot run, which is what makes an era-typical Win32 binary
  emulable at all.
- **Synchronise your debugger with Ghidra rather than transcribing addresses.** `ret-sync`
  bridges WinDbg/x64dbg/GDB to Ghidra and handles ASLR rebasing; 12.1 also ships x64dbg
  synchronisation, and `Debugger-agent-dbgeng`/`-x64dbg` are in the box. This is what turns
  "the user can verify in-game" from an anecdote into an observation landing on an exact
  address. **WinDbg TTD** goes further: a recorded trace answers *"what wrote this field"*
  definitively, which no static witness in this document can. (Two corrections to the
  folklore: `revsync` has **no** Ghidra support, and `ghidra_bridge` is superseded now that
  Jython is gone.)
- Runtime is the right instrument when static evidence is *exhausted*, not merely
  inconvenient. Phrase the question as a specific breakpoint and byte range before
  reaching for it.
- **Record a runtime observation at its own provenance tier**, never as binary ground
  truth: it is a fact about one execution, one build and one configuration. It is strong
  evidence and it is not `IMPORTED`.

---

### A naming source can be EXHAUSTED program-wide — measure that once and record it

Round zero's table says to record measured zeros. This is the follow-on: a source that *did* yield
names can later have nothing left, and that fact is worth establishing globally rather than
rediscovering per round.

**Measured on one binary.** The "self-announce" source — debug/diagnostic strings of the form
`ClassName::MethodName`, which the developers emit for their own logging — had previously named two
classes that nothing else reached. Asked globally, it holds **62 strings naming 13 distinct
classes, and all 13 are already known**. The source is not weak; it is *finished*. Likewise the
export route: of the class-shaped type names appearing inside mangled export symbols but not yet
claimed, all were value types and subsystems (`CLVector`, `CMatrix44`, `CSoundSystem`), not
hierarchy nodes.

Recording "exhausted, as of this measurement" costs one query and saves every future round the same
hopeful detour. It is also honest in a way "not yet tried" is not: a stale unknown reads as an
opportunity forever.

### When a round prices out, ask what the blocked population has in COMMON

A refuted round is cheap only if you read what refuted it. **Measured:** a naming round for 81
classes died on two measured zeros — no source could name their methods. The autopsy found a shared
property nobody had looked at: **every one of the 81 overrode the same vtable slot**, and an earlier
phase had already established what that slot is. Sweeping that property across the whole program
found **224** classes with the same shape, not 81, and three independent witnesses for the claim.

The instance was a naming problem with no evidence. The pattern was a naming problem with ground
truth behind it. Before abandoning a blocked population, enumerate what its members share — a slot
index, an allocation size, a call site, a base — and ask whether any of those is already decided.

### The IMPORT TABLE is a naming source, and grouping by it groups by mechanism, not purpose

A stripped binary still tells you the name of every function it borrows from another module. Those
names came out of the **file**, are self-documenting, and cost nothing to collect — on a target with
no debug information this is often the strongest evidence tier available, and it is routinely left
unused because it looks like plumbing rather than like recovery.

Two things it does well:

- **It bounds a subsystem mechanically.** Sweeping for bodies that call any of a middleware's imports
  turns "the audio code, wherever that is" into a finite list. Measured: 26 `_AIL_*` (Miles Sound
  System) plus one MCI import bounded a game's audio layer to **exactly 9 unnamed bodies**, all of
  which could then be read.
- **The uniqueness form is far stronger than the membership form.** "This body calls
  `_AIL_start_sample`" is weak when several do. "**This is the only body in the program that starts
  a sample**" nearly names the function on its own, is mechanically checkable, and fails loudly if a
  later change splits the body. Prefer sole-caller-ship wherever the population is small enough for
  it to hold, and assert it rather than assuming it.

And the trap, which is what makes this a rule rather than a tip:

> **A shared API is a shared MECHANISM. Purpose has to come from the body.**

In the same sweep, **2 of the 9 "audio" bodies were copy protection** — they reach the CD through
`mciSendCommandA`, the same multimedia API the music does, and what they actually do is XOR-decrypt
a file with a fixed key, close any CD-player window, and read the disc's table of contents. Naming by
import group would have shipped two `Sound*` names onto a disc check. Let the import table decide the
**candidate set** and never the name; read the body for the verb.

The same caution applies in the other direction to library-vs-game separation above: a game function
that wraps one middleware call is game code, and a middleware routine statically linked into the
binary calls nothing imported at all.
