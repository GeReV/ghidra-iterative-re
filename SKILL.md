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

**Confidence convention.** Claims marked *(doc)* come from Ghidra's own documentation but
have not been executed. Unmarked claims were either measured live or are project history.
Quoted numbers from one binary are examples for calibration, not properties of yours.

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

**Each stage, concretely:**

| Stage | What it means |
|---|---|
| **checkpoint** | `checkpoint.py`-style: end transaction, save, snapshot, version-control step |
| **apply** | Write only this round's certainty tier, tagged `SourceType.AI`, gated on an `apply` argument |
| **cascade** | `analyzeChanges` after telling `AutoAnalysisManager` precisely what changed |
| **verify invariants** | Diff the whole-program invariant and `raise` on change |
| **harvest** | Read evidence back out, **excluding `SourceType.AI`**, into versioned evidence files |
| **adjudicate** | Turn raw evidence rows into *decided* layout/ownership records: apply the confidence rule (how many structurally independent witness kinds agree), resolve conflicts between witnesses, and **record what you deliberately excluded and why**. This is the step that produces a `confidence` column. Excluded evidence must be written down — a later reader re-deriving from raw rows will otherwise reach a different answer and think yours is wrong. |

### Cascade is the stage that gets skipped — build a forcing function

**Measured on this project: an agent that knows this loop still skipped `cascade` on two
consecutive applies.** Nothing failed, nothing warned, and the round looked complete. The
damage was subtle and specific: the "did the decompiler improve?" measurement was taken
against a program that had **never been re-analyzed**, so a stated finding rested on an
unverified read. (Re-running the cascade afterwards happened to confirm the finding — which
is exactly why the omission was invisible.)

It is the easiest stage to skip because it is the only one with no artifact. `apply` writes
symbols, `verify` raises, `harvest` writes CSVs — but `cascade` produces nothing you can
look at afterwards and notice missing. So make its absence visible:

- **Every mutating script ends by notifying `AutoAnalysisManager` what changed and calling
  `analyzeChanges`** — or prints, explicitly, why no cascade applies (a repo-CSV-only
  apply, for instance). Silence is indistinguishable from forgetting.
- **Never measure the payoff of an apply before the cascade has run.** "I applied types and
  the decompiler looks the same" is not a finding until the analyzer has seen the change.
- **Bracket the cascade with the same invariant as the apply.** `analyzeChanges` is the
  operation most able to cause silent collateral damage, and re-analysis can *undo*
  curation — so also re-assert that what you applied is still there. Checking function
  count alone would not notice a re-analysis that cleared every struct you just applied.
- Record it in the round's metrics like any other stage, so a round with applies and no
  cascade is visible in the history rather than only in memory.

**One round is not one session.** A round is a unit of *work* — it can span sessions, and a
long round should be split across fresh contexts to avoid the naming drift described under
failure modes. The loop's termination test ("two dry rounds") is about the work, not about
how long any one agent context lives. Prefer many short executions inside one round over
one long execution spanning several.

**Quantify every round.** Emit a metrics row per checkpoint into an **append-only** file,
with internal-consistency checks that **raise, not warn** (counts that must sum, sets that
must be disjoint). A workable schema: `label`, `timestamp`, `program_version`,
`functions_total`, then one column per population you are tracking (identified library,
unclassified, types applied, evidence rows), so a row is a complete snapshot and any two
rows are comparable. Document each metric's definition in its own file and record renames —
definitions drift, and a metric silently redefined between rounds makes the whole history
incomparable. This is what makes "did the round accomplish anything" answerable rather than
a matter of impression.

**Track replay debt honestly.** Keep a written record of which steps are *not* yet
reproducible from a clean state and why. A pipeline nobody has replayed end-to-end is
untested, however green each step looked when it ran.

**Emit a C header from recovered types.** It is the bridge to a reimplementation, it is
diffable outside Ghidra, and generating it forces every recovered layout to be complete and
self-consistent rather than approximately right. **Ghidra has this built in** —
`ghidra.app.util.exporter.CppExporter` (the File → Export Program → C/C++ mechanism), whose
`CPPResult` exposes `headerCode()` and `bodyCode()`. Use it or a deliberate subset of it;
hand-rolling a header emitter is the exact failure this skill warns about elsewhere.

**Apply in certainty order, never convenience order.** Ground truth from the binary
first, then mechanical derivations from it, then inferences. A round that applies a guess
before an available certainty has corrupted every round after it.

Ghidra's own advanced course says of the **Decompiler Parameter ID** analyzer: *"If you
run this analyzer too early or before fixing problems, you can end up propagating bad
information all over the program."* The cascade propagates errors as well as facts.

## The trust model — the load-bearing part

Ghidra tracks provenance natively via `SourceType`. Its real priority order is
**`USER_DEFINED` > `IMPORTED` > `ANALYSIS` = `AI` > `DEFAULT`** — note the tie:

| SourceType | Means | Usable as evidence? |
|---|---|---|
| `USER_DEFINED` | A human decided this | Yes, but record that a human is the source |
| `IMPORTED` | Created by the **importer** from the file itself: export-table labels, debug records | **Yes — ground truth** |
| `ANALYSIS` | Anything an analyzer produced — **including the demangler's output** | Only with the producing mechanism named |
| `AI` | *"content produced through AI assistance"* — **yours**. Ghidra ranks this **equal to `ANALYSIS`**, not below it | **NEVER** (see below) |
| `DEFAULT` | `FUN_`/`DAT_` placeholders | No — absence of information |

**"`AI` is never evidence" is a policy you impose, not something Ghidra enforces.**
`SourceType.AI` and `SourceType.ANALYSIS` have *equal* priority — verified live:
`AI.getPriority()` is 2 and `AI.isHigherPriorityThan(ANALYSIS)` is `False` in both
directions. So Ghidra will let an AI-tagged symbol win against analyzer output exactly as
often as the reverse, and any command that arbitrates via `isHigherPriorityThan` treats
them as peers. The tier is a *label you can filter on*; the discipline has to live in your
harvesters.

**Do not assume demangled names are `IMPORTED`.** Measured on a PE with 981 mangled
symbols: only **6** of 2640 functions carried `IMPORTED`, while **985** carried `ANALYSIS`.
The raw export label is importer-created ground truth; the *demangled name applied to the
function* is analyzer output one inference removed from it. Census your program before
building a trust rule on a tier.

**Tag every mutation you make `SourceType.AI`.** The anti-circularity rule then stops
being a discipline and becomes a query filter:

> **Harvesters must exclude `SourceType.AI` symbols from evidence.**

A round's own applications then structurally cannot become evidence for the claim that
motivated them. Verify the filter fires: tag one known symbol `AI`, confirm it drops out
of the harvest.

**The filter is needed before a harvester's *second* run, not its first** — which is why
it is so often missing. Written before any apply exists, a harvester is correct; it turns
circular the moment it is re-run after one, with nothing in it having changed. See
"Harvesting traps" for the measured case and the assertion that catches it.

### `SourceType` is necessary but not sufficient

**Confirmed live in this project: `SourceType` cannot distinguish two different naming
sources that share a tier.** A genuine PE-export demangled name and a name harvested by a
string heuristic both carry `SourceType.ANALYSIS` — indistinguishable by tier alone. So
the tier tells you *how trustworthy the class of source is*, not *which source it was*.

Consequences:

- Keep **per-address evidence files** recording which mechanism produced each name, and
  consult them before any tier-based inference.
- **Never infer provenance by elimination.** "Not default-named and not in a library
  namespace, therefore a real export" is a landmine: on the next regeneration it silently
  relabels heuristically-derived names as binary ground truth, destroying the very
  distinction the column exists to carry. Cross-reference the evidence files first and
  emit an explicit `unknown` for anything that matches nothing.
- That fix immediately surfaced a real mislabel worth knowing generally: **a thunk
  inherits a plausible qualified name from its target while its own local symbol carries
  `SourceType.DEFAULT`.** Elimination logic called those exports; they are thunks. This is
  detectable as a census discrepancy — here, 1649 functions at `DEFAULT` against 1646
  `FUN_`-prefixed names, and the difference of 3 was exactly the thunks.

**Census the tiers before designing around them.** One read-only pass counting
`symbol.getSource()` over functions *and* over all symbols costs nothing and tells you
which tiers are actually populated, whether the tier you plan to trust is the one carrying
your ground truth, and whether name-based and tier-based counts disagree (they did here, by
exactly the thunks). `SourceType.AI` is script-usable and orders as documented — verified:
priority 2, equal to `ANALYSIS`, below `IMPORTED` and `USER_DEFINED`.

### Mutate through scripts, not MCP tools

**Audited in GhidrAssistMCP source (`symgraph/GhidrAssistMCP@master`): every mutating tool
hardcodes `SourceType.USER_DEFINED` — 18 occurrences across `RenameSymbolTool` (9),
`VariablesTool` (3), `StructTool` (3), `CreateFunctionTool`, `SetFunctionPrototypeTool`,
`CreateDataVarTool`. `SourceType.AI` appears nowhere in the codebase.**

An MCP mutation therefore launders your inference into the *highest* provenance tier,
indistinguishable from a human decision and permanently uncleanable from evidence. **Do
all mutation from scripts.** MCP read-only queries are fine. Audit any other MCP server
the same way before trusting it.

> **Revisit:** GhidrAssistMCP issue **#66, "Consider SourceType.AI for mutation tools"**
> (<https://github.com/symgraph/GhidrAssistMCP/issues/66>) is **open** and proposes exactly
> this change, reaching the same conclusion independently. Check its state before assuming
> the hardcoded behaviour is still current — once the server can emit `SourceType.AI`,
> mutation through MCP becomes viable and this rule relaxes to "verify what tier the tool
> writes."

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

- **The sharper version of that trap: the second run harvests *your own* output.** Stale
  evidence is the mild failure. The severe one is a harvester that re-reads names *this
  agent applied* and files them as findings — because those names then feed whatever
  decides the next apply, closing the loop on itself.

  It hides well for a structural reason: **a harvester written before any apply exists is
  correct when written, and only becomes circular the first time it is re-run after one.**
  Nothing changes in the harvester, no test fails, and the output looks richer than before.
  Measured here: re-running a slot-symbol sweep after a naming apply would have folded 8
  agent-applied names back into the very file feeding the calibration gate and the naming
  rule — i.e. into the inputs deciding what to name next.

  So the `SourceType.AI` exclusion is not a property you add once to "the harvesters" — it
  is a property every harvester needs *before its second run*. Audit them at the moment the
  round's first apply lands, not when they are written. Treat an agent-sourced name as
  **identical to no name at all**: absent from evidence, and still an open candidate. Then
  assert it: after any re-harvest, check that every address in your applied-names ledger
  comes back unnamed in the fresh evidence, and fail if one does not.
- **Distinguish measured-zero from structural-zero.** "I looked and found nothing" and
  "there was nothing to look at" are different findings. Emit counters that separate them.
- **A short name is not an identity.** Demangled short names collide across a hierarchy —
  an override shares its base's name, so two different tables can look identical. Join on
  **addresses**, not names.
- **Population counts depend on the symbol filter, silently.** In this project
  `getExternalFunctions()` returned 134 imports, not the expected 136, because two of them
  exist only as `SymbolType.Label` and not as `Function` symbols — so that API cannot see
  them at all. Verify a population before building on its size.
- **A hand-written seed list can contain structurally inert entries.** Of 28 marker
  imports seeded into a call-graph closure, 2 were unreachable by the mechanism consuming
  them, so the effective seed set was 26. Check that each seed is actually visible to the
  API that will consume it, and report the effective count, not the written one.
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
2. **Debug information, if any survives — check this before anything else, because it
   dominates everything below it.** A shipped or leaked `.pdb` (Windows), DWARF, `.debug`
   sections, or a `.map` file gives you names *and* full types *and* often line numbers
   outright, collapsing most of this list into a single import. Ghidra has first-class PDB
   support (`ghidra.app.util.bin.format.pdb`, and the install ships `docs/README_PDB.html`
   describing the parser); DWARF has a full loader plus **debuginfod** download support as
   of 12.1. Rarely present in retail game builds — but the payoff is so much larger than
   every other source that checking costs nothing by comparison. Look for a `.pdb` path
   string, `RSDS`/`NB10` signatures, and a CodeView debug directory entry. Also check
   whether the *modding community* has one; leaked symbol files circulate for popular games.
3. **Exported and mangled symbols.** Mangled C++ names carry class, member, virtualness,
   and full signature. Even a stripped retail build often exports more than expected.
   **Data exports name GLOBALS, with static types** (`?pRendEng@@3PAVCRendEng@@A` ==
   `CRendEng *pRendEng`) — parse the PE export directory for non-code targets, don't
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
   the cheapest naming route rather than meaning "look harder."

**Use the developers' vocabulary.** Terminology from the game's own UI, manual, and
strings is what the authors called these things. Name recovered entities to match it, not
to match your model of the game.

### Run the cheap test for each before budgeting work on it

Every item above is a *hypothesis about your binary*, and each has a one-command test.
Run the whole column first: a measured zero is a finding, and it stops you planning a
phase around a source that isn't there. The right-hand column is what these returned on
one 1999-era MSVC/x86 retail game binary — illustrating that yields differ wildly and
must be measured, not assumed.

| Source | Test | Example yield |
|---|---|---|
| MSVC RTTI | strings `.?AV`, `type_info` | **0** — built `/GR-`, closing the cheapest naming route |
| Exported vftable symbols | strings `??_7` | **14** authoritative class→vtable pairs |
| Virtual / multiple inheritance | strings `??_8`, `??_9` | **0** — plain single inheritance |
| Assert source paths | strings `.cpp`, `.c`, `.h`, drive-letter prefixes | **1** `.cpp`, **1** `.h` — source root only |
| Mangled export symbols | count symbols starting `?` | **981**, carrying class + virtualness + full signature |
| Data exports (globals) | PE export directory entries outside `.text` | **73** typed named globals; the singletons (`pRendEng`, `pWorld`, …) and the allocator family |
| Companion-DLL linkage | shipped DLLs' import tables naming the exe | **2 renderer plugins** importing 46/47 exe symbols — the whole renderer API vocabulary |
| Embedded scripting VM | strings `lua`, `Lua_`, registration-table shapes | not yet run |
| Config / CVar parser | strings for known setting keys, `.ini`, `.cfg` | not yet run |
| Middleware / engine | version banners, known import sets | Miles Sound System and DirectInput identified |
| Debug/PDB residue | `.pdb` paths, `RSDS`/`NB10` signatures, `.debug` sections | not yet run |
| Localization / string tables | large contiguous string blocks, id→string arrays | not yet run |

Record every result, including the zeros, and distinguish **measured-zero** ("searched,
absent") from **not-yet-run**. Conflating those is how a project convinces itself a source
was exhausted when it was never tried.

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
- **A missing FID match is silent by construction.** There is no "almost matched"
  signal to inspect, so absence of a hit is not evidence of absence of library code.
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

## Dispatch recovery — virtual calls are one case of several

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
  constant after assignment (issues #650, #516). But three explicit mechanisms do work,
  escalating: name the slots via a vftable struct, mark the table `CONSTANT` so the
  decompiler reads *through* it, or override the call site with
  `RefType.CALL_OVERRIDE_UNCONDITIONAL`. The last one **writes your inference into the call
  graph** — tag it `SourceType.AI` and never let a harvest read it back as fact.
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

## Platform breadth

Console and pre-modern targets are common in game RE, and three hazards there invalidate
assumptions the rest of this skill makes. Per-architecture detail:
**`references/platforms-eras.md`**.

- **Get endianness right at import, before anything else.** The same processor family runs
  big-endian on some consoles and little-endian on others — PowerPC is BE on GameCube, Wii,
  Xbox 360 and PS3; MIPS is LE on PS1/PS2/N64/PSP; SH-2 on Saturn is BE. Picking the wrong
  language variant (`PowerPC:BE:32` vs the LE variant) silently garbles the entire
  disassembly, and no amount of later analysis recovers from it. It is the cheapest possible
  mistake to make and the most expensive to notice late.
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
- [GhidrAssistMCP source](https://github.com/symgraph/GhidrAssistMCP) — audited for `SourceType` usage; [issue #66](https://github.com/symgraph/GhidrAssistMCP/issues/66) proposes `SourceType.AI`
- [LLM Agent-Assisted RE with Quantitative Readability Metrics](https://arxiv.org/html/2606.06838) — metric gaming, coverage, context rot
- [Agentic RE: Building Custom AI Skills with Coding Agents, Recon 2026](https://cfp.recon.cx/recon-2026/talk/SHYHKM/)
- [und3rf10w ghidra-scripting agent skill](https://skillsmp.com/creators/und3rf10w/ai-ghidra-tools/plugins-ghidra-skills-ghidra-scripting)
- [Beginners Guide to Reverse Engineering (Retro Games)](https://www.retroreversing.com/tutorials/introduction) · [Reverse Engineering a GBA Game](https://www.starcubelabs.com/reverse-engineering-gba/) · [Reverse engineering save game files](https://mytechnologyblog532.wordpress.com/2016/11/13/reverse-engineering-save-game-files/) · [Unity IL2CPP save file RE](https://blog.painite.ch/en/blog/unity-save-file-reverse-engineering/)
- This project: `notes/tooling-capabilities.md`, `notes/assertion-discipline.md`,
  `notes/phase2-report.md` §5
