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
