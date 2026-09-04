# Applying changes — what to apply, and how to survive it

*Reference for the `ghidra-iterative-re` skill. Only `SKILL.md` is loaded when the
skill is invoked; this file is read on demand when its trigger fires. New lessons of
this kind belong here, not in `SKILL.md`.*

**Read this when:** you are about to mutate the program, emit a C header from recovered
types, or decide which apply is worth the round.

**In this file:**

- What actually improves decompilation
- Emit a C header, and add the assertions yourself
- Mutation safety — the rest of the bracket

## What actually improves decompilation

- **Function signatures.** Parameter/return types propagate to call sites — the strongest
  real cascade. Mangled names encode full signatures;
  `DemanglerCmd(addr, mangled).applyTo(program, monitor)` applies name *and* signature —
  and, with default options, **disassembles and can create a function** at the address
  (`DemanglerOptions.doDisassembly` defaults to `true`). It is a mutating operation of the
  same class as the ones you bracket, not a naming convenience.

  **But measure the VARIANCE before you commit one, because the Parameter ID analyzer got
  there first.** "Signatures propagate to call sites" is true and it is not a licence: a
  committed signature's payoff is that it replaces N *different* per-site guesses with one
  answer, so the round is worth exactly what N is — and where the Decompiler Parameter ID
  analyzer has already run, N is often 1. Measured on one binary, on the two highest-fan-in
  functions in it (a custom arena allocator and its free, 852 call sites): both already
  carried an analyzer-committed `__cdecl` one-parameter prototype at
  `SignatureSource=ANALYSIS`, structurally identical to the one the planned round proposed,
  and decompiling all 428 callers and inspecting every CALL p-code op found **361 of 361**
  allocator sites already rendering one argument and `int *`, and **475 of 475** free sites
  already rendering one argument and no result. Zero variance on either axis; the round
  evaporated on one read-only probe. Count the distinct (arity, return type) pairs across the
  call sites first — that number *is* the payoff.

  Two traps ride along with it, both measured in that round:
  - **The more CORRECT type can be the less USEFUL one, and the summary line cannot show
    it.** `void *` is right for an allocator and `int *` is a decompiler guess — but every
    one of those 361 sites *consumes* the return, and `int *` renders a dereference directly
    where `void *` forces a cast at each use. Committing the correct type would have cost
    legibility at 361 sites for nothing measurable, while the round's own headline ("852 call
    sites given a committed signature") would have read as success. Grade a type change by
    what it does at the call sites that exist, not by which type is more nearly true.
  - **An `AI` signature does NOT outrank an `ANALYSIS` one, so the cascade may take it
    back.** Verified from the enum on a 12.1.2 install: `ANALYSIS` and `AI` are both priority
    **2**, `isHigherPriorityThan` false in *both* directions. Laying an AI-tagged signature
    over an analyzer-committed one is a peer overwrite that `analyzeChanges` is entitled to
    reverse — so check `getSignatureSource()` on the target *before* planning the apply,
    not after the cascade eats it.
- **Defining functions at undefined code.** Feeds analyzers new material, extends
  pointer-table runs truncated by undefined targets, adds call-graph edges. Often the
  highest-leverage single mutation.
- **Type vftable struct components as `Pointer` → `FunctionDefinition`, not as bare
  pointers.** The idiomatic build — `ClassUtils.getVftDefaultEntry(dtm)` — returns a plain
  `PointerDataType`, which names the slots but types nothing, so every virtual call still
  decompiles as `(*(code *)(*(int *)this + 0x48))()`. Build a `FunctionDefinitionDataType`
  from each slot TARGET's function instead and the same call renders
  `(*p->vftable->GetPos)(p)` with typed arguments and a return type that propagates into
  the caller's locals. Measured on one binary: 1053 of 1068 slots across 15 classes, from
  363 distinct definitions (slots share targets through inheritance), every one
  `__thiscall` with a real prototype. Two things make it cheap and safe: take the
  signatures only from targets carrying a MANGLED symbol, so the evidence tier is the
  binary's own string; and `Structure.replace(ordinal, ptr, 4, name, comment)` is a
  1-for-1 swap, so assert the struct's length and component count are unchanged and the
  blast radius is confined to decompilation. The slots you cannot type are usually the
  compiler-generated destructor thunks, which have no export by construction — leave them
  bare rather than inventing a signature.
- **Struct layouts.** Field accesses become named. Can *change* signatures as a side
  effect: a large struct returned by value switches to the hidden return-storage-pointer
  convention, so `T Func(this)` becomes `T * Func(this, T *__return_storage_ptr__)`.
- **Data mutability.** *"The decompiler will display the contents of a memory location if
  the contents are marked as constant. Otherwise it will display a pointer to the
  location."* Settings: normal / constant / volatile / writable — per-data, or
  per-memory-block via the Memory Map. Marking genuinely read-only sections constant is
  factually correct, cheap, and high-leverage. Marking hardware registers **volatile**
  stops the decompiler folding away repeated reads.

### Emit a C header, and add the assertions yourself

**Emit a C header from recovered types, and add the assertions yourself.** It is the bridge
to a reimplementation, it is diffable outside Ghidra, and generating it forces every
recovered layout to be complete and self-consistent rather than approximately right.

Emit the *types* with the built-in: **`ghidra.program.model.data.DataTypeWriter(dtm,
writer)`** is the public, supported ANSI-C type emitter ("The ANSI-C code should compile on
most platforms"). Do **not** reach for `CppExporter.CPPResult` — it is a **private nested
record**, uncallable from a script and absent from the javadoc, and `CppExporter` itself
decompiles the entire program as a side effect (options `CREATE_C_FILE`,
`CREATE_HEADER_FILE`, `EMIT_TYPE_DEFINITONS` — Ghidra's typo, not ours). It delegates its
type emission to `DataTypeWriter` anyway.

Then wrap that output yourself with `_Static_assert(offsetof(T, field) == K)` per field and
a `sizeof` assertion per class, and **compile it**. No Ghidra exporter emits compile-time
offset assertions, and the compile is the only step that recomputes offsets from the C
object model rather than from your own arithmetic — which is precisely the step worth
having. Hand-rolling the *type* emitter is the failure "Check for a built-in before writing
your own" (`references/api.md`) warns about; hand-rolling the *assertions* is the whole point.

**And emit FIXED-WIDTH types, not `char *` or `long`.** The target's word size is not the
host's, and getting this wrong silently relays every field after the offender. It breaks the
header, the recompile and the reimplementation — in that order of discovery and the reverse
order of cost. See **Target ABI vs host ABI** in `references/oracles-and-abi.md`.

### Mutation safety — the rest of the bracket

- **A scalar invariant is a smoke alarm, not a change set.** Keep the function count as the
  cheap canary, then get the real measurement: check the program in before the round, and
  afterwards run `ProgramDiff(checkpointProgram, currentProgram).getDifferences(
  ProgramDiffFilter(CODE_UNIT_DIFFS | SYMBOL_DIFFS | FUNCTION_DIFFS | REFERENCE_DIFFS |
  COMMENT_DIFFS), monitor)` → an `AddressSetView`. That enumerates categories a function
  count is *structurally* blind to — references, comments, register context, source map,
  equates, bookmarks. **`ProgramMerge`** then restores a *specific* category from the
  checkpoint (`mergeFunctions`, `mergeLabels`, `mergeCodeUnits`, `mergeComments`,
  `mergeReferences`, `replaceFunctionSignatureSource`), which is the closest thing to the
  undo that `canUndo()` refuses to give you for a previous script run's transaction.
  Version control works in a local, non-shared project, and Program Differences can diff
  against a version picked from the Version History table.
- **Scope address sets as narrowly as the operation allows.** Broad set + large archive
  is the dangerous combination; the incident did not reproduce narrowly.
- **`Function.setName(sameName, SourceType.X)` IS A NO-OP FOR THE SOURCE — and a mutation
  that silently does nothing is indistinguishable from one that worked.** Measured: a round
  re-tagging 22 project-applied names from `ANALYSIS` to `AI` called `setName` with each
  function's existing name and the new tier. Ghidra short-circuits when the name is
  unchanged, so no tier moved. All 22 calls returned cleanly, the census printed, the ledger
  was written, and the run would have reported success. `Function` has **no source setter at
  all**; **`Symbol.setSource(...)` is the API** — `dir()` on the live objects shows
  `getSignatureSource`/`setSignatureSource` on `Function` and `getSource`/`setSource` on
  `Symbol`. Two things generalise past the specific call:
  - **Count the population you claim to have changed.** Per-item calls that do not throw are
    not evidence; only a before/after count of the property itself is. Here
    `ai_symbols == before + N` was the sole check capable of noticing, and the same shape
    catches any API that silently declines.
  - **Ask the object rather than recall the API.** One five-line probe printing the
    SourceType-related members of `Function` and `Symbol` settled in seconds what reasoning
    from memory had already got wrong once.
- **A RAISED RUN LEAVES YOUR LEDGER AHEAD OF THE PROGRAM.** The ledger append precedes the
  verification, so a failed apply claims names it did not apply — and the ledger is what your
  provenance gates join against, so the damage lands in the guard rail rather than in an
  artifact. Revert the raised run's rows BY NAME before re-running.
- **BEFORE WRITING INTO A SLOT, ENUMERATE ITS READERS — AND ANSWER WITH A NUMBER, NOT A
  COMPATIBILITY ARGUMENT.** Comments, tags and bookmarks feel inert, and they are not: measured,
  a round adding provenance PLATE comments at every project-named address was writing into the
  exact slot `Ghidra`'s Function ID analyzer uses, which a standing gate read and regex-parsed a
  `Libraries:` block out of. The tempting move is to reason that a prepended block separated by
  a blank line cannot disturb a parser looking further down — and that reasoning was *correct*,
  which is precisely why it is not evidence. What shipped instead: a census counting the overlap
  (**1** address of 3618), a writer that prepends and preserves rather than replaces, and a
  before/after equality check on all **56** parses. Any slot with a reader needs the equality
  check; grep for who reads it before deciding it is inert.
- **RUN EVERY GATE AT THE NEW VERSION — "this round could not have touched it" is sound
  reasoning and still an assumption.** Measured: a state line was drafted claiming five gates
  green after a comment-only round when two had been re-run and two were inferred safe on the
  grounds that comments cannot move types or vtables. True, and it costs two minutes to find
  out. The most-read sentence in a project's records should carry measurements, not deductions.
- **A PRECONDITION MEASURED BEFORE A BATCH OF OPERATIONS CAN BE INVALIDATED BY THOSE
  OPERATIONS — re-check it immediately before the operation it guards.** Measured: an
  applier verified at census time that no type of a given name existed; a *merge step in the
  same run* then created one, because moving a function into a class namespace makes Ghidra
  materialise a placeholder struct of that name; the next step died on a duplicate-name
  exception with the round half-applied. A census is for deciding SCOPE. It is not a
  substitute for the check at the point of use.
- **WRITE A MULTI-STEP APPLY TO CONVERGE ON THE END STATE, NOT TO ASSUME THE START ONE.**
  `canUndo()` is `False` for a previous script run's transaction and your snapshot-restore
  path is probably untested, so a raise part-way through leaves a state you must be able to
  RE-ENTER. Make every step report "already" and do nothing when its end state holds; then a
  re-run on a finished program is a no-op that still verifies every intended fact, and
  forward recovery becomes provable rather than hopeful. Two things make it work:
  - **Derive the population from a REPO fact, not a PROGRAM fact.** Membership taken from an
    append-only ledger's last row per address survived the partial apply intact; membership
    read from "who currently occupies this namespace" would have made the round's scope
    depend on how far the failed run got.
  - **Give the applier a dry arm that runs the REAL code path** (`do=False` through the same
    convergence functions). That is the arm that catches this class of defect; a selftest
    built only from constructed inputs cannot.
- **`replaceDataType(placeholder, keeper, True)` MOVES the keeper into the DISCARDED type's
  category.** Measured: structs folded over demangler-created placeholders ended up under
  `/Demangler/`, not at the root where they had been, and every consumer looking up
  `"/" + name` silently reported them ABSENT. When you fold types, enumerate the consumers
  of the type's PATH, not just of its name.
- **AN INVARIANT BRACKET THAT CHECKS A *DELTA* GOES BLIND THE MOMENT A RUN RAISES.** Measured:
  an apply died part-way, and the next run's before/after delta compared two states that were
  *both* already wrong — the delta was clean and the program was damaged. Pin the bracket to an
  **absolute** expected value, not to "the same as when I started". The cost is that the pin must
  be re-set whenever you legitimately change the number, and that is the point: **a re-pin is a
  decision with a reason attached, a range is a decision to stop noticing.** Measured again two
  rounds later — a probe stopped on `datatypes=1396, expected 1395` after an apply that created
  exactly one array type. The temptation is `>=` or a tolerance; the correct move is to account
  for the delta from the apply's own record (one `created` row at that version boundary) and
  re-pin to 1396 with the reason in the comment. A range would have silently absorbed the next
  apply's collateral damage.
- **AN APPLIER THAT REBUILDS FROM AN ARTIFACT WILL DISCARD EVERY REFINEMENT THAT LIVES ONLY IN
  THE PROGRAM — AND NOTHING WILL NOTICE.** Measured: a re-run would have replaced a 268-field
  embedded record, applied by a later round directly into the program, with the one opaque
  `uint8_t[300]` the artifact still carried. The struct still tiles, its length is unchanged, and
  a bracket counting functions, symbols and datatypes sees nothing — the discarded type is still
  in the DataTypeManager, merely unreferenced. **Diff each already-final class against its own
  plan before writing, and hold back any class the program types more specifically.** Two traps
  inside that guard, both paid for:
  - **Compare RESOLVED types, not names.** The first version compared `getName()` and called every
    `dword` component richer than the plan's `uint32_t` — the same type under two names — so every
    already-applied class came back held-back for a reason unrelated to any refinement. It was
    right for two classes and wrong for a third: a check passing for the wrong reason.
  - **Decide what "more specific" means, explicitly, because that predicate is load-bearing in
    BOTH directions.** Placeholder shapes (`undefined*`, `uint8_t[N]`) must not count as richer.
    That one line is what later let the same guard handle the *opposite* case — an artifact that
    had grown FINER than the program — without modification: the coarse cells being replaced were
    placeholder-shaped, so they offered no resistance, while the genuine refinement did. **The
    guard was right for a case nobody designed it for, which is worth writing down**: the next
    edit to that predicate silently decides whether re-applies still work.
- **APPLYING AN OPAQUE `uint8_t[N]` OVER A REGION DOES NOT MERELY FAIL TO HELP — IT DEGRADES THE
  EVIDENCE NEEDED TO SUBDIVIDE THAT VERY REGION.** Measured: filling a class tail with a byte
  array made the decompiler re-render two dword stores as twelve one-byte stores, and one-byte
  cells across the harvest went 15 → 109. The round that applied it understated its own cost,
  because "the placeholder is neutral until we learn more" is the natural assumption and it is
  false. Two consequences: **take widths from the INSTRUCTION, never from the decompiler's
  varnode**, and expect any later attempt to subdivide that cell to be arguing against evidence
  your own apply corrupted. It is reversible — subdividing the cell later restored the same
  accesses to single dwords, with the instruction width identical throughout, which is the proof
  of which rendering was the false one.
- **A "NOTHING CHANGED" DETECTOR MUST COMPARE WHAT THE RUN WOULD WRITE AGAINST WHAT THE PROGRAM
  HOLDS — NOT A PROXY LIKE SIZE.** An idempotent applier needs to distinguish a genuine re-run
  (where before == after by construction, so a payoff delta is *unavailable* rather than *zero*)
  from a real change. Measured: the test was "is every target already at its final length", which
  cannot see a class at the same size whose contents the artifact has since subdivided. It would
  have reported a real, measured payoff as *"UNAVAILABLE ... not a gain this run produced"* —
  the same lie of form the unavailable branch exists to prevent, pointed the other way. Ask the
  question of the classes the run actually WRITES: one it deliberately holds back says nothing
  about whether anything changed. **When you change such a rule, print both the old verdict and
  the new one permanently** — that is the two-step diff baked in, and it costs one line.

### An applier's population rule does not follow the evidence; somebody has to move it

**Measured: 137 classes had no applied struct, and the cause was one gate keyed to the wrong
source.** The applier accepted a class only if a *naming* registry announced it. That was correct
when written — the naming registry was the layout source. The layout evidence later came from a
different witness entirely (constructor writes, 925 of 1046 rows), and **the gate did not follow**.
Over the 137: size decided **137/137**, base already laid out **137/137**, namespace 54/137,
declarer **0/137**.

Nothing warned, and nothing could: **a population rule that reaches nothing looks exactly like a
population that has nothing in it.** Both print zero and both leave the artifact empty.

- **The tell is a gate whose pass rate is 0 while every other gate on the same population passes
  ~100%.** Print every gate's pass count as a fraction of the population, always, including the
  ones you expect to pass — a single `ALL FOUR: 0 of 137` line beside four per-gate lines locates
  the binding constraint in one read.
- **Ask it of every applier you own, periodically and not only when a round stalls:** is this gate
  keyed to the evidence it applies, or to whatever source happened to exist the day it was written?
- The reason this survives so long is that the *stalled* population is invisible from inside the
  applier. It is only visible by joining the applier's gates against the population the evidence
  now covers — which is a read-only query nobody runs, because the applier reports success.

### Every mutating applier needs a state meaning "already exactly what I would write"

**Measured: an apply raised part-way — after writing all 19 types, before the cascade — and could
not be restarted.** Undo was unavailable (`canUndo=False`; the script provider's transaction was
not rolled back on the exception). The applier's state machine had three states: `absent`,
`placeholder`, `applied` — and `applied` meant *"somebody else's, refuse to touch it"*. So the 19
types the round had just written were indistinguishable from another round's, the route flipped to
`already_applied`, and the population guard would have raised on the disagreement. **A partial
failure is precisely when a resume is needed, and a three-state machine cannot offer one.**

The fix is a fourth state — *this is already exactly what I would write* — verified rather than
rewritten. Note it is not a weakening: it checks the shape in full (one component, right offset,
right field name, right type, right length) before granting it.

- **A sibling script in the same repo had the missing arm from the start.** Before designing a new
  applier's state machine, read the states of the most similar existing one; the expensive states
  are the ones somebody already learned to need.
- **Pin the census to the ROUTE population, not the number of writes.** A human approved "19". On
  the resumed run 0 needed writing; pinning to writes would have demanded `apply 0` and silently
  broken the tie to what was approved. The route population is 19 on both runs.
- **A raised run is not a no-op run.** Check what actually landed before deciding how to recover —
  here, querying one type settled it in one call, and the answer (not rolled back) determined the
  whole recovery path.

### Before choosing where an annotation lives, count how many of the population can carry it

**Measured: the obvious home for "this class adds no data members" is the type's own description,
and 96 of the 101 classes had no type at all.** That route would have recorded the finding for 5 of
101 and printed a clean summary — coverage theatre with no bug in it, because every write it
attempted would have succeeded.

Pick the anchor that **all** of the population has (here the dispatch table's address, which every
class has by construction), and state the coverage as a fraction before writing anything.

### Never destroy an existing annotation; append below it, and count the three cases separately

An address that already carries a comment usually carries *provenance* — which round named it, on
what evidence. Overwriting is silent and unrecoverable. Distinguish and count: **fresh** (nothing
there), **ours-refreshed** (drop your previous block by marker, re-add — so a re-run cannot
accumulate copies of itself), **appended below foreign text** (kept intact). Print all three; a
round that expected `fresh` and got `foreign_append` has learned something about the address space
before it writes.

### A naming apply creates TYPES you did not ask for

**Measured.** A round that applied **no types at all** — it named 203 functions into 98 newly
created class namespaces — moved the program's whole-datatype count by **+136** and changed a
`signature` column in an artifact three steps downstream.

The mechanism: creating a class namespace materialises a 1-byte placeholder *type* of that name,
and the cascade then types a `__thiscall` function's `this` against it. So "this is a naming round,
the type invariants cannot move" is false, and a round that reasons that way will be surprised by
its own gate.

Budget for it: **a naming round that creates namespaces should expect the datatype bracket to
fire**, and should plan the by-name account for the placeholders before running, not after. The
payoff is real and worth having — `this` typed at the class is what turns raw pointer arithmetic
into member access — but it is a type change arriving through a name-shaped door.

### When an apply's payoff disappoints, ask whether it is the wrong apply or the wrong ORDER

**Measured, on the same script run twice.** A struct apply over ~100 classes retyped **7 of 40**
functions and said so. Two rounds later the *identical* script, artifact and population rule
retyped **107 of 140**. Nothing about the apply changed; in between, a naming round had attributed
203 functions to those classes, and **a struct only retypes where a signature already points at
the class type**.

Both readings of the first result would have been wrong. "The apply is not worth it" would have
discarded a 15x payoff sitting one round away. "The measurement is pessimistic" would have been
dressing up a real number. What was right: apply anyway (the work was correct and the artifact was
needed), **state the small number plainly**, and record what the payoff was gated on — which is what
made the multiplier visible when it arrived.

The generalisation: an apply's benefit is often a product of two applies, and the one you are
holding may be the second factor. Before concluding a route is low-value, ask what its payoff is
*conditional on*, and whether that precondition is cheap and already on the backlog.

### Predict whether a round moves the type count, and be suspicious when the bracket disagrees

Two rounds, opposite shapes, measured:

| round | what it applied | datatype count |
|---|---|---|
| naming into new namespaces | **no types at all** | **+136** — `createClass` materialises a 1-byte placeholder per namespace, and the cascade types `__thiscall` `this` against it |
| structs over existing placeholders | **81 structs** | **0** — `replaceDataType` swaps in place under the same name |

So the intuitive rule is exactly backwards, and both surprises are cheap to avoid by predicting the
number before the run. A bracket that fires when you expected silence — or stays silent when you
expected a delta — means the apply did something other than what you believe, and that is worth
stopping for rather than re-pinning past.

### A derived type that is the same size as its base ties with it on every size-keyed join

**Measured.** Representing "this class adds no data members" as `struct Derived { Base base; }` is
correct and drift-free — and it necessarily creates a type of *exactly* its base's size. Do that
100 times and every witness that identifies a type BY SIZE gains candidates: a 128-byte block-copy
site went from two possible types to five, the three new ones being classes that had been 1-byte
placeholders the round before.

The new candidates are genuinely that size, so this is **dilution, not error** — but it is
permanent, it is invisible unless you diff the artifact, and it is a property of the design rather
than of any one round. Record it where the size-keyed witness is documented, and require later
rounds that lean on such a join to state the candidate count as a fraction rather than quoting it
as if it were narrow.

## Before reverting a raised apply, establish WHICH SIDE IS AHEAD

The standing repair for an apply that raised is *do not promote its output — revert the artifacts it
touched, by name.* That rule is right, and it encodes an assumption worth making explicit: **the
artifacts ran ahead of the program.** Which is the normal failure, because artifacts are written
last, after the program mutation has already succeeded.

There is a second shape, and the standing rule makes it worse rather than better.

**Measured.** A script renamed five symbols, rewrote both its side ledgers, then raised on a mistyped
cascade call. Program and ledgers had moved **together** and agreed with each other exactly; only the
cascade and the post-cascade re-assert had not run. Reverting the ledgers there would have
**manufactured** the desync the rule exists to prevent — turning a consistent, unfinished state into
an inconsistent one.

- **Diff the program against the ledger before deciding.** The correct repair is whichever action
  restores agreement. Sometimes that is finishing the operation, not undoing it.
- **The rule is about CONSISTENCY, not about undoing.** "Revert on raise" is a heuristic for the
  common case; the invariant underneath it is that the program and its ledger must agree.

## An applier that REWRITES rows must be convergent from the start

Most appliers append: a row is written once and never touched again, so a half-finished run always
leaves the artifact ahead of or behind the program in a detectable way. An applier that **rewrites**
existing rows is different — a partial run can leave both sides consistent, and then re-running is
the natural repair rather than a hazard.

So write it to converge: **a target that already carries the intended value is a corroboration, not
a conflict.** Skip it and continue. This is the same three-way guard a good applier already uses to
tolerate a name it applied in an earlier session (*present and correct → skip; present and different
→ raise; absent → apply*), and it costs three lines.

Written that way, the incident above would have been a re-run rather than a diagnosis. Written the
other way, a raise leaves an operation that **cannot be completed except by hand** — which is how a
careful project ends up hand-editing a ledger.

### An applier whose population rule is the condition its own apply DESTROYS cannot converge

The previous section says every mutating applier needs a state meaning *"already exactly what I
would write"*. Here is the way that state can be present, correct-looking, and **unreachable by
construction** — with nothing in the code to show it.

**Measured.** An applier promoted a ground-truth symbol to primary at every address whose primary
was a placeholder. Its population rule was, in full, *"an address carrying an authoritative symbol
whose primary is placeholder-shaped."* It applied cleanly over four addresses, every guard passed,
the invariant bracket held, the post-cascade re-assert held. A bare re-run then raised
`VACUITY: 0 shadowed addresses` **on its own successful output** — because promoting the real name
is exactly what stops an address being shadowed. The apply had consumed its own population.

The three-state machine was there. `already` could never be entered: an address in that state no
longer satisfies the membership predicate, so it is not a candidate to be classified.

- **Reading the applier does not reveal this.** The branch is syntactically reachable and
  semantically dead, and every review of the code says it handles the re-run case. What reveals it
  is *running the applier a second time* — which is the argument for a bare re-run being a standing
  step after every apply rather than a nicety.
- **The fix is the rule this file already states, pointed at a new symptom:** derive the population
  from a **repo** fact. Here, the union of the program-derived set and the human-approved census.
  That rule was written for partial-failure resume; it is the same fix, and noticing that saved
  designing a second one.
- **The vacuity guard has to move with the rule.** *"Zero candidates is a rule fault"* is true only
  before the round runs. The check that survives an apply is *"zero to write AND zero already"* —
  otherwise the guard that protects you from a broken rule becomes the thing that fails on success.

Ask it of any applier whose predicate mentions the state it writes: **if this runs to completion,
does its own selection still find these rows?** A predicate over *what is missing* answers no.

### A print is not a write

**Measured.** A mutating applier ended with
`print("LEDGER: append these rows to <ledger>")` and left the append to a human. The apply landed
two agent-tagged symbols the ledger did not carry — precisely the both-directions provenance
invariant the project's gates exist to catch. Nothing in the applier failed; it reported success,
and the discrepancy existed for exactly as long as it took someone to read the output and act on
it. Had the output been long, or the session ended, the gate would have caught it later and the
round would have looked like collateral damage rather than an unfinished step.

**An applier that instructs a human to finish it has not finished.** The ledger append belongs
inside the same run as the mutation. Two details make that safe:

- **Write it convergently** — skip a row already present for this key — so a re-run cannot
  accumulate duplicates.
- **Expect the fix to expose a second defect in the apply itself.** Making the ledger re-runnable
  forces the *program* side to be re-runnable too, and there the trap is that
  `createLabel`/`setName` with a value already present is a **silent no-op**. A delta bracket of
  the form `symbols == before + N` then raises on a program that is already correct. Count what
  the run actually writes, not what its census contains.

### Creating a function makes its whole body visible to every function-walking sweep at once

**Measured.** One function created at previously-unclaimed bytes changed **four** committed
artifacts. Two were expected (the symbol inventory, and the string inventory's "referenced from no
function" cell). One was a bonus corroboration. The fourth was a survey of virtual-dispatch sites,
which gained **three** rows — dispatch sites that had always been in those bytes and were invisible
only because the bytes belonged to no function.

**Budget the artifact churn of a function-creating apply against the BODY, not against the one row
you meant to add.** Every sweep in the project that iterates defined functions now sees an
additional body's worth of instructions, and each will report whatever it is built to find there.

The upside is real and worth expecting rather than being surprised by: two of those four changes
were *corroborations arriving free* — an independent sweep reported the new body has the structural
shape its claimed role requires, which is a witness nobody had to design. And where the sweep names
functions, check the change shows your name **filtered out**: seeing the agent-name filter visibly
suppress your own fresh name in a regenerated artifact is the anti-circularity rule being exercised
rather than merely present.

### When a gate refuses your row, change the row's HOME before you change the gate

**Measured, twice in one round, and both refusals were right.** A round wrote a recovered 16-byte
type into the program and then had to record it. The first artifact it tried was the project's
decided-types file, on the strength of an earlier round's note saying *"adding a row here is an
approved step"*. The type gate raised: every row in that file is joined against a harvested
evidence file which the project may not regenerate, and the type in question had been discovered
in a later phase with no such evidence. The second was the field-type-upgrade file, on a precedent
that matched the situation exactly in substance — an exported member accessing a block cell at
element width, with an in-body element-count bound. That gate raised too: its guard verifies
`element_size * N == span` **from the ctype string**, so it accepts only scalar element names and
cannot size a struct.

Widening the second guard's pattern was one line. It would have been wrong, and the reasons
generalise:

- **A guard in a producing sweep is a rule, and changing it owes the full two-step diff and its
  own poison.** That cost is worth paying when the rule is wrong. It was not wrong; it was simply
  built on a mechanism (string arithmetic) that cannot express this case.
- **It was unnecessary.** A third committed artifact — the one whose stated purpose is that *"a
  rebuild-from-artifact is only safe if every decision lives in an artifact"* — already resolved a
  ctype's base through the type manager and built arrays from it, so it accepted the row unchanged.
- **The test to run first:** is there another committed artifact whose consumer already takes this
  row as written? If yes, the guard is not the problem and the row is in the wrong file.

Two smaller things fell out of the same round:

- **A note that authorises an artifact write grants the PERMISSION and does not settle the HOME.**
  *"Adding a row to X is an approved step"* is a statement about sign-off. Whether the row belongs
  in X is a separate question with its own answer, and the gate is the thing that knows it.
- **Verify the retreat.** After reverting both rejected appends, the check that the withdrawal was
  clean rather than approximate is that both files are **byte-identical to the base revision** and
  appear in no diff. A revert that leaves a stray blank line or a re-ordered row is a content
  change nobody adjudicated.

### A report that cannot see the effect it claims to measure is worse than no report

**Measured.** An applier ended with a section printing each affected function's signature, to show
the payoff: a 16-byte by-value return should switch to the hidden return-storage-pointer
convention. It printed `getPrototypeString(...)`, which omits both `this` and the hidden return
pointer — so it rendered the identical string before and after, while the committed symbol
artifact recorded the parameter count going **1 → 2** with the storage pointer added. The round's
genuine result read as a non-result, and was only recovered by diffing the artifact.

- **Pick the accessor by what it is documented to include**, not by which name reads best. Where a
  compiler-introduced parameter is the thing you are looking for, walk the parameter list (which
  includes auto-params) rather than a formatted prototype string.
- **When a predicted effect appears not to have happened, suspect the instrument before the
  prediction.** A silent report and a real non-event are indistinguishable from inside the script,
  and one cheap artifact diff separates them.

### Predict the datatype delta with an account BY NAME, before the run

Same round: predicted `+1` and got `+1`, where the naive reading says `+2`. The apply wrote a
struct **and** an array of it, but the struct already existed as a demangler-created 1-byte
placeholder and was **edited in place**, so only the array was new. That is knowable before
running — check whether each type you are about to write already exists — and it converts the
bracket from something you re-pin after the fact into something that confirms your model of what
the apply does.

### An option's label and its description must make the same promise

**Measured, and the failure is in the asking rather than the answering.** A census option was
presented to a human as *"add a shared typedef too"* while its description offered only *"record
in the artifact that these elements share a domain"* — a type versus a note, with the more
attractive wording on the label. The human chose quickly, then came back to ask what the typedef
had been. That is how the mismatch surfaced at all.

The whole point of approving a census **to the row** is that the approval is informed. A label
that promises more than its description delivers defeats that, and no amount of accuracy further
down repairs it: **the label is what gets read.**

- **Write the label at the same specificity as the description.** If the label names a mechanism
  ("typedef", "retype", "rename"), the description must propose that mechanism.
- **Treat a fast answer as a reason to re-read your own option, not as evidence it was clear.**
- When the discrepancy is found, put the choice again and say plainly that the first framing was
  wrong — do not quietly deliver the more attractive reading on the grounds that it was chosen.

## An applier needs a FORWARD recovery, not only a rollback

The rule *"an applier whose post-check can raise must ship its recovery"* is usually read as *ship a
revert*. Half the failures need the other direction. Measured: an applier mutated 23 functions, ran
its cascade, **passed its first post-check**, and then crashed in the *second* on a variable-name
clash with the scripting bridge — leaving the program correct, verified, and ahead of its ledger.
Reverting a correct mutation in order to re-apply it identically is churn, not safety.

So ship a **`finish` mode**: re-run every post-check and append the ledger, mutating nothing. Two
details make it trustworthy:

- Take its control-group baseline from the **committed artifact**, not from an in-memory capture —
  the artifact survives the crash, the capture is exactly what was lost.
- It must still be able to *fail*. `finish` re-asserts the applied state rather than assuming it; a
  half-applied program must not be ledgered as complete.

The same round supplies the reason the snapshot must exist at all: **write the BEFORE state to the
artifact before the first mutation**, never to a local. Here it was already on disk when the crash
happened, so nothing was lost.

## A gate that REGENERATES artifacts can revert a repair between adjudication and commit

Measured, and it cost a commit. A round repaired a defect in a committed artifact, diffed it, and
adjudicated the change. The stability gate — which regenerates most committed artifacts to prove they
are reproducible — then ran, **silently overwrote the repaired file with the defective output**, and
the round committed that. The commit undid its own fix. It surfaced only because the gate was re-run
*afterwards* and compared against the commit.

**Rule: a green gate run before `git add` says nothing about what was actually staged. Re-run any
artifact-regenerating gate AFTER committing, and diff against the commit.**

## A harness that EXECs source files must open them with an explicit encoding

The root cause above is worth its own line, because it is invisible and it breaks the one property
such a harness exists to guarantee. The gate loaded each producer's source with a bare
`open(path).read()`. The disassembler's own script provider decodes scripts as **UTF-8**; a bare
`open()` uses the platform's locale default, which on a Windows host is **cp1252**. A single non-ASCII
character in a producer's source therefore arrived mangled, and the producer — correctly encoding its
output as UTF-8 — wrote it back **double-encoded**.

Two consequences. **The same script produced different bytes down the two execution paths**, which is
precisely the byte-identity property the gate was built to test. And an encoding check over the
artifacts could not see it: double-encoded text is *valid* UTF-8, so a gate asserting
**decodability** passes on it forever. **Assert what you mean: decodable is not correct.**
## A pre-mutation snapshot must be APPEND-ONLY, or the second round destroys the first round's recovery

The rule *"capture the BEFORE state into the artifact before mutating"* has a failure mode that shows
up on its second use, not its first.

Measured: an applier wrote its pre-mutation snapshot with an overwriting write. Re-pointed at a later
round's three targets, it would have replaced the previous round's twenty-three rows — that round's
*only* revert source — with three, **while executing the very rule that exists to make recovery
possible**. The safety mechanism was about to erase its own history, and the dry run looked perfect
because the snapshot it wrote was correct for the rows it knew about.

Make it append-only and give every row the round that wrote it; make `revert` filter on that column.
Then backfill the existing rows with theirs, so the file is consistent rather than half-labelled.

**The general rule: any artifact whose purpose is recovery is append-only and self-identifying. An
overwriting safety mechanism is not one.**


## Look for the smaller change before you accept the blast radius

A queued item can be right about the risk and wrong about where the change has to go, and the
warning about blast radius is what stops anyone looking for a cheaper site.

Measured. A project's construction-body scanner tracked "which registers provably hold the object
pointer at offset 0" — a set several sweeps, appliers and diagnostics all read. Member sub-objects
are constructed through `LEA ECX,[this+K]`, which by design never enters that set, so a chain walk
driven by it could not follow them and a large fraction of every class's storage went unwitnessed.
The backlog scoped the fix as *"make the set offset-aware"*, correctly noted that it was shared by
a dozen consumers, and required a two-step old-rule-vs-new-rule diff with the old rule run live.

The set did not need changing. The same function already carried a **parallel** alias tracker
resolving `reg -> this + delta` for an unrelated side channel, and the call branch already ran
before the caller-saved registers were cleared — so the offset could simply be *read* at the site
the old rule was rejecting. What shipped was a new key appended at an existing site, with a
perturbation surface that was **empty by construction**; the two-step diff then confirmed that
rather than discovering it.

So: when a change looks like it must touch shared machinery, **enumerate what the function already
computes before you change what it computes.** A side channel added for one purpose is often the
capability the next round needs, and the cost of checking is one read of the file.

## Document what a rule cannot witness, beside the rule that makes it sound

The same module explained, correctly and at length, *why* the offset-0 discipline is sound —
member constructors receive `LEA ECX,[this+K]`, a class with its own vfptr cannot embed a member at
offset 0, therefore the set is trustworthy. What it never said is the consequence: that every byte
a member constructor writes is invisible to any consumer driven by that set, **by construction**.

The cost of the omission was not a wrong number, it was a *misread* one. The producer emitted a
3,152-byte region marked `unknown` — it had seen the region and could not name it — and downstream
that read as "nobody has looked here yet" rather than "this instrument cannot look here". Four
months, and a class scoring 7% coverage that nobody re-opened.

A soundness argument names a blind spot whether or not it says so. **Write the blind spot down in
the same comment**, in the vocabulary a coverage report uses, so the honest `unknown` in the
artifact can be traced back to the rule that guarantees it.

## A deferral is only honest if something can find it again

It is often right to apply markup that is correct but incomplete — a struct with an undefined
region, a field typed by width and not by meaning, a layout flattened from an ancestor because the
immediate base's extent is unknown. Claiming less than you know is the whole discipline. But
"we'll revisit this when we know more" is a promise that decays the instant the round ends, unless
the revisiting is mechanised.

The cheap mechanism is a **worklist artifact**: one committed row per deferred decision, carrying
the identity of the thing deferred, the exact region affected, and what was used instead.
Measured: 21 structs were built from an ancestor rather than their immediate base because three
base sizes are unrecoverable; each skip became a row of `(class, skipped base, gap lo, gap hi,
prefix actually used)`. When one of those sizes later becomes measurable, the affected structs are
a **query**, not an archaeology exercise — and the applier's state machine already refreshes a
flattened prefix whose source has changed, so the rebuild is a re-run rather than a new round.

Three properties make it work, and the third is the one usually missed:

- **The row names the region, not just the class.** "This class is approximate" is not actionable;
  "bytes [840, 872) of this class are undefined because sizeof(X) is undecided" is.
- **The row names what was used instead**, so a later reader can tell a deliberate substitution
  from an omission.
- **The artifact is covered by whatever checks your committed artifacts** — reach model, encoding
  gate, stability canary. An uncovered worklist is a text file that silently stops being
  regenerated, which is the same as not having written it.

An undefined region plus a sentence in a commit message makes exactly the same claim about the
binary and none of the claims about the future.

## Split an apply by RISK CLASS, so stronger evidence cannot launder weaker

When a round has several changes to make and they all pass their checks, the efficient-looking
move is one apply with one census. Resist it whenever the changes rest on **different witnesses**.
A single census makes the population look homogeneous, and if one subset later proves wrong there
is no way to attribute the failure — the strong evidence and the weak evidence shipped together
under one number.

Measured. 76 signatures were repaired in one round and deliberately applied as two:

- **58 `__fastcall` → `__thiscall`.** Both conventions pass argument 1 in the same register, so
  the relabel is *abi-identical* — it renames what the code already does. The witness is a
  liveness probe showing the second register carries nothing.
- **18 `__stdcall` → `__thiscall`.** This *adds* a parameter the signature never modelled, which
  is a claim **about** the ABI rather than a relabelling of it. The witness is entirely different:
  sibling implementations of the same vtable slot dereferencing the object register.

Two censuses, two approvals, and a **checkpoint between them**, so the program has a named version
in which the safe half is applied and the risky half is not. If the second half ever has to be
reverted, the revert target exists and is not entangled with the first.

The test is not "how confident am I" but **"if this subset turned out to be wrong, would I be able
to tell which evidence failed, and could I undo it alone?"** If the answer to either half is no,
it is a separate census.

## When one operation makes two claims, say which one carries the risk

Some applies are a single call that asserts two independent things. Setting a `__thiscall`
convention with an empty formal parameter list, for instance, does a *convention relabel* and a
*parameter retype* at once, because the disassembler then derives the implicit `this` from the
function's parent class. Written up as "we relabelled 78 signatures", the round looks uniform. It
is not:

- the **relabel** is ABI-identical — both conventions pass argument 1 in the same register — so its
  only claim is that the *second* register carries nothing, which register liveness decides
  mechanically and independently;
- the **retype** rests entirely on the parent namespace naming the right class, and on a binary
  with no RTTI and no debug records that namespace is *your own prior inference*.

Separating them is what tells you which witness the round actually needs. Once written down, the
question "what corroborates the namespace, other than us?" has an obvious answer — the set of
classes whose vtable holds the body, which comes from the binary — and it stops being optional.

**And make that rule DIRECTIONAL rather than exclusive.** The tempting form is "a body appearing in
several classes' vtables means the attribution is unreliable, refuse it." That is wrong: a base
method appearing in 21 subclass vtables *is* the defining class's method — the slot is inherited,
not ambiguous. The correct form is *the claimed owner must be an ancestor-or-self of every class
whose vtable holds the body*. Measured, the directional rule kept 10 rows the exclusive one would
have discarded while still refusing 4 that genuinely contradicted the namespace.

Two mechanical notes that cost a guard each:

- An empty formal list means the tool injects the implicit parameter **and nothing else**, so a
  body that really takes further stack arguments has them silently DELETED. No count notices —
  functions, symbols and types all stay put while the prototype quietly narrows. Refuse any target
  with more than one modelled parameter rather than truncating it.
- Re-assert **after** the cascade. An AI-tier signature is exactly what the demangler and the
  analyzer are free to overwrite, since those tiers rank equal.

## A metric moving the WRONG WAY can be the recovered layout finally telling the truth

After applying a type, expect some quality metrics to get worse, and read the mechanism before
calling it a regression.

Measured. Typing 78 `this` pointers moved the good numbers hard — index expressions through the
wrong type **998 → 0**, member accesses through the object **4 → 1708**, named (non-placeholder)
fields **988 → 1725**, with 78 of 78 bodies improving. In the same measurement, *undefined locals*
over those bodies **rose 167 → 334**, 40 bodies worse against 5 better.

Both easy write-ups were wrong. "Cascade noise" is refuted by the control group, which held at
190 → 190 across the same re-analysis. "The apply degraded the decompilation" is refuted by the
body itself: `undefined1 *puVar1 = &this->field_0xf0;` — the new undefined locals come from taking
the address of struct fields **not yet recovered**. Before the apply the decompiler had no struct
to consult and *guessed* a type from usage; afterwards it propagates the layout's own honesty about
which bytes are still unknown.

So the number is real, it is caused by the apply, and it is not a regression: it is the recovered
type refusing to invent what it does not know. **The worsened bodies become a query for the next
field-recovery round** — they name exactly the classes whose layouts have the most unrecovered
span. Report it at the same weight as the wins; a round that only reports the metrics that moved
its way is choosing its own scoreboard.
