# Harvesting traps

*Reference for the `ghidra-iterative-re` skill. Only `SKILL.md` is loaded when the
skill is invoked; this file is read on demand when its trigger fires. New lessons of
this kind belong here, not in `SKILL.md`.*

**Read this when:** you are about to write or re-run a sweep, a probe, or anything that
reads evidence back out of the program — and before you believe a census, a zero, or a
reach estimate it produced.

**In this file:**

- Self-harvest and circular evidence
- Derivations, and repairing them at the source
- The instrument you read with
- Censuses, zeros and skip counters
- Calibrating and pricing a witness
- Undefined code and the byte-pattern engine
- Identity, joins and populations

### Self-harvest and circular evidence

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

  **The oldest harvester is the most dangerous one**, and it is worth naming that explicitly
  because it inverts the intuition that old code is safe. The sweep that ran first, before
  any apply existed, is the one whose author had no reason to write the filter — so it is
  both the most trusted artifact in the project and the only one with no protection. Measured
  on a second occasion here, years of rounds later: the original vtable sweep was regenerated
  and **578 of 578** changed function names joined, *by address*, to the agent's own applied-
  names ledger — 0 from any other source. The committed artifact was found to have already
  captured 8 such names from an earlier round. **Join by address, never by name**, or the
  audit itself inherits the collision problem below.
- **A producer whose CLASSIFICATION INPUT is derived from its own output is self-harvesting,
  there is no `SourceType` tier to filter on, and you fix it at the DERIVATION rather than by
  loosening the gate.** The self-harvest rule above is about
  *symbols*; this is the same failure on the **artifact** axis, where the only defence is
  arithmetic. Shape: sweep S writes artifact A; derivation D folds A into artifact B; S reads
  B to decide what is new. Measured: a destructor-chain witness classified its claims against
  a hierarchy file built from its own edges, and on the second run its two contradictions had
  become agreements *with itself* — agreement counters up by exactly the folded edges
  (100/209 → 101/210), contradictions **2 → 0**, and every gate green for the wrong reason.
  Three things do it, and the third is the one that actually catches it:
  - **Design the artifact so the pre-fold view is RECONSTRUCTIBLE.** Record, per row, what the
    value used to be (`superseded_base`) and which entities the fold created. A fold you
    cannot undo on paper is one you cannot audit.
  - **Subtract your own previous output at load, then classify — and PRINT that you did it.**
    A silent subtraction is as unauditable as a silent fold.
  - **The vacuity guard is what catches this, not any value check.** Nothing here was
    "wrong": no count mismatched, no artifact drifted. The only thing that fired was the
    selftest arm asserting there was still a contradiction available to poison with. A
    self-harvesting producer fails by having its findings quietly go to zero while every
    other number *improves*, so a census selftest without a vacuity arm cannot see it.
- **N-OF-N AGREEMENT IS WORTH NOTHING WHEN ONE SIDE WAS APPLIED FROM THE OTHER — AND IT IS THE
  MOST PERSUASIVE-LOOKING NUMBER YOU WILL PRODUCE.** Measured: a probe found **43 of 43** names
  in a property registrar matching the field names on the recovered structs exactly. Read as an
  independent witness confirming the layouts, that is a striking corroboration. It is
  tautological: the applier that built those structs *reads that registry*. This is the
  self-harvest rule on the ARTIFACT axis, where no `SourceType` tier exists to filter on, so the
  only defence is provenance and arithmetic — before recording any agreement as evidence, ask
  **"would this cell hold this value if we had applied nothing?"** and name the producer on each
  side. A perfect score is the shape to distrust first.
- **"NO EVIDENCE SOURCE EXISTS FOR THIS" IS A RESULT, AND BELONGS IN THE RECORD AS ONE.** A
  layout round can recover offsets and widths for thousands of cells and still have no source
  anywhere that could NAME them. Measured: of every committed artifact carrying both an offset and
  a name, exactly one had a class column intersecting the recovered population at all — reaching
  16 of 198 classes, whose names were already applied. So 182 classes and ~3750 fields have no
  naming source in the project. Left as a task on an open list, "name the fields" reads as
  available work forever and gets re-derived every few rounds; recorded as a measurement, with the
  sources checked and what would change the answer, it stops costing anything.
### Derivations, and repairing them at the source

- **Two identical branches of an `if`/`else` are a silent UNDER-REPORT, not dead code.** Where
  you wrote a branch to express a distinction and both arms do the same thing, the distinction
  is simply unmeasured — and the output still looks like a working measurement with a small
  answer. Measured: a probe meant to treat "the class's own store survived" and "it was
  elided" differently emitted the same pairs in both cases, so an entire claim category went
  untested and single-element chains produced nothing at all. Repairing it moved that arm from
  unmeasured to 209 agreements and 1 contradiction. Diff the arms of any `if`/`else` you wrote
  for a reason.
- **A calibration gate on a witness that may legitimately STRENGTHEN must be a FLOOR, not an
  equality.** The failure worth catching is the witness going *quiet*; an equality additionally
  fails every time the program legitimately grows, which trains the reader to re-baseline it.
- **Do not MERGE the artifact you are using as an independent cross-check.** A new witness
  agreeing with an unrelated artifact 30 of 30 is worth more as a standing corroboration than
  as 29 extra rows — folding them makes the two agree by construction and destroys the
  measurement that justified the witness in the first place. Independence is a property you
  spend when you merge.
- **A pipeline that derives wrong and patches afterwards violates this rule from the
  inside, and it hides for years.** The version everyone catches is a one-off hand edit.
  The version nobody catches is a *second script* that corrects the first one's output as
  a routine pipeline step: the committed artifact is correct, every consumer is happy, and
  the derivation stays broken. Measured: a vtable sweep labelled each table by its slot-0
  function's namespace and a separate script overwrote that with honest labels afterwards
  — so a plain run of the sweep produced a file naming **29746 of 31341 rows after a
  single base class**, 95% of it wrong, and the stability harness (which runs sweeps, not
  post-steps) reported ~29700 drifting cells on every pass until someone read them. Test
  for it directly: **run each producer alone and ask whether its output is the committed
  artifact.** If a second step is required to make it honest, the artifact is not
  derivable and the fix belongs in the producer — after which the post-step should be
  retired to a no-op you assert stays a no-op.
- **Fix a wrong artifact at its DERIVATION, not by patching the rows.** When a defect in
  committed evidence is proven, the tempting round is to amend the data — it is smaller,
  and the diff is reviewable. But a patched artifact stops being re-derivable, which is the
  property that made it evidence; the producing rule stays wrong; and the next regeneration
  silently reintroduces the whole defect. Measured here: a boundary rule that mistook a
  constant-propagated virtual-call reference for a table start had forged 7 tables. Patching
  the 55 affected rows and fixing the rule cost the same effort — the rule fix also fixed
  the *next* regeneration, and turned the 7 known cases into a live calibration gate.
- **Separate program drift from the change under test, or you cannot read the diff.**
  Re-running a harvester that was written many rounds ago never reproduces its committed
  output, because the program has moved underneath it — so "regenerate and diff" conflates
  two entirely different populations of change. Run the **old** rule first and diff *that*
  against the committed artifact (pure drift, adjudicated alone), then diff the **new** rule
  against the old rule's fresh output (your change, and nothing else). Keep the old rule
  reachable behind an argument for exactly this. Here the naive one-step diff showed ~29748
  rows changed and was unreadable; the two-step version resolved it into "a post-processing
  step I had forgotten to apply", "578 rows of self-harvested names" (above), and finally
  the **55 rows** the change was actually supposed to make.
### The instrument you read with

- **A p-code harvester must state its simplification style, and calibrate across two —
  but read `buildDefaultGroups()` before choosing which two.**
  `DecompInterface.setSimplificationStyle` takes `decompile` (default), `normalize`,
  `firstpass`, `paramid`, `register`. The default style runs a rule called **`earlyremoval`
  that deletes COPY operations whose output has no descendant — including stack writes
  rendered dead by a function signature, whether or not that signature is correct.** For a
  project whose layout evidence is "which offsets does this constructor write", that would
  be a silent under-count arriving from inside the tool, possibly caused by a signature
  *you* applied.

  **An earlier revision of this document said to compare `normalize` against `decompile` to
  expose it. That is wrong, measured against a 12.1.2 install**, and it is worth stating
  loudly because the advice looks obviously right:

  ```
  coreaction.cc:5563   actdead->addRule( new RuleEarlyRemoval("deadcode") );
  ```

  and `ActionDatabase::buildDefaultGroups()` lists `"deadcode"` in the member arrays of
  **both** `decompile` and `normalize`. That comparison holds the named rule constant. The
  only shipped styles omitting `deadcode` are **`register`** (`base`, `analysis`, `subvar`)
  and **`firstpass`** (`base` alone) — so `register` is the style that actually tests
  `earlyremoval`, and `firstpass` is the control that shows what the analysis was buying.
  What `normalize` really omits is **`typerecovery`**, which is a different and often more
  interesting question: it is the group that consumes *the types you applied*.

  **Generalise past the specific fact:** `buildDefaultGroups()` is the ground truth for
  what a style runs. The javadoc's one-line descriptions ("omits type recovery and some of
  the final clean-up steps") do not say which rules move, and a round planned from them
  targets the wrong comparison — burning the round's whole budget on a comparison that was
  never capable of firing.

  Measured on one project, over the 14 array-loop candidates behind its only exact size
  witness: `decompile`, `normalize` and `register` each decided all **12** strides with
  **identical values**; `firstpass` decided 1. Zero contradictions, zero cases of a
  less-processed style deciding where the default declined. A real answer, and cheap — but
  only because the comparison was picked from the groups rather than from the prose.

- **An instruction-level harvester that accepts only IMMEDIATE operands under-counts exactly
  where the compiler HOISTS a repeated constant into a register — and the blanks look like a
  property of the data.** A string-registration parser read `MOV [reg+8], imm` and missed
  `MOV EDI, "float"` loaded once per body and stored from the register per record: 49 of 83
  type cells came back blank, every one of them `float`, every `int` captured — a pattern
  that invites a story about the API ("only ints are typed") rather than about the reader.
  Before theorising about a blank census, compare ONE body against the decompiler's view of
  it; then fix it at the derivation, keep the old arm reachable behind an argument, and run
  every consumer through the old-vs-committed / new-vs-old two-step diff (here: 0 drift, then
  exactly 4 evidence cells and 0 decided-layout rows). Track register loads with the same
  invalidation discipline as an alias tracker: any write to the register drops it, a CALL
  drops the caller-saved set, the callee-saved registers survive — which is precisely why the
  compiler parks the hoisted strings there.
- **Measure the decompiler-derived SURFACE before auditing it.** The audit above was queued
  on the premise that the project's layout witnesses were harvested from decompiler output.
  One census — *which scripts construct a `DecompInterface`* — refuted it: every write-set
  witness read raw instructions through `listing.getInstructions()`, and the entire
  decompiler-derived evidence surface was **one function**. A p-code simplification rule
  cannot perturb a witness that never asks for p-code. The premise had sat unchallenged in a
  queued round for a week, and it cost one command to check.

  **QUALIFIED, and the qualification is the more useful half: "not the decompiler" is not the
  same as "not p-code", and there are THREE levels here, not two.** Reading raw instructions
  does insulate a witness from simplification rules — and if it reads
  `getDefaultOperandRepresentation()`, it buys that insulation by throwing away the
  instruction's SEMANTICS, which is a worse trade than it looks:

  | level | what it is | carries direction? | simplification rules? |
  |---|---|---|---|
  | rendered operand text | `getDefaultOperandRepresentation()` + a regex | **no** | no |
  | **raw instruction p-code** | **`Instruction.getPcode()` — the SLEIGH spec itself** | **yes** | **no** |
  | decompiler p-code | `HighFunction` / `DecompInterface` | yes | yes |

  The middle row is the one projects skip, and it is strictly better than the first: same
  insulation, plus the processor specification's own account of what the instruction does.
  Measured: a write-set witness parsed operand 0 with a regex and recorded it as a WRITE,
  because operand 0 is the destination for `MOV`. It is a **source** for `CMP`, `TEST`, `FLD`,
  `FMUL` and every x87 memory form — so **1552 of 21249 committed `ctor_write` rows (7.3%),
  across 151 of 198 classes, were reads**. Nothing failed; the offsets were real, the sizes
  they bounded were right (a read of `this+K` proves K is inside the object exactly as a store
  does), and only the LABEL was false. `getPcode()` containing a `STORE` op answers the
  direction question authoritatively, because it *is* the processor's definition.

  Two riders, both measured on that repair:

  - **Do not hand-write the mnemonic list.** The first attempt at the fix was a
    `READ_OP0 = {CMP, TEST, PUSH, FLD, ...}` set typed from memory. Graded against the SLEIGH
    answer on the same instructions it **disagreed on 70 of 2221**. This is the skill's own
    "discover the API from the install, not from memory" rule pointed at instruction semantics,
    where it is easier to fall for because the mnemonics feel like common knowledge.
  - **A relabel moves evidence in BOTH directions — measure both before predicting the net.**
    The repair was expected to *reduce* corroboration, since some cells were corroborated by a
    write that turned out to be a read: 8 such rows were counted, and a drop was predicted.
    The actual result was a **rise, 317 → 347**, because the mislabel had been *collapsing two
    independent witness kinds under one name* and hiding corroboration at far more cells than
    it invented it at — **570 cells gained** a distinct kind against **41** that lost one. The
    prediction was wrong because only the loss direction had been counted, which is exactly the
    half-measurement this document demands a counter for when a rule changes.

- **Do NOT test whether a witness depends on types you applied by withholding an opcode
  downstream of type recovery.** Same project, same round: the exact-size witness read
  multiplier constants from `INT_MULT` inputs and `PTRADD` element sizes, and `PTRADD`'s
  constant is the size of a pointed-to type — i.e. of a struct the project itself applied.
  Withholding `PTRADD` lost **9 of 12** strides, which reads as "9 exact sizes rest on our
  own applied types". **False.** `RulePtrArith` (group `typerecovery`) *consumes* the
  `INT_MULT` it folds into the `PTRADD`, so withholding the `PTRADD` removes the only
  surviving copy of a constant that was never type-derived; at `normalize`, with
  `typerecovery` off, the identical constants return as `INT_MULT` and all 12 decide. Turn
  the **type recovery** off and re-measure. An opcode-withholding test is invalid wherever
  the analysis consumes the alternative form — and note which way it failed: it
  *manufactured* a serious finding rather than hiding one.
- **Your own markup perturbs similarity witnesses.** Ghidra's BSim tutorial warns that
  applying debug information changes BSim signatures and can degrade matching — so a
  correlation run *after* an apply round is not measuring the same thing as one before it.
  Capture similarity evidence early, or record which program version it came from.
### Censuses, zeros and skip counters

- **Distinguish measured-zero from structural-zero.** "I looked and found nothing" and
  "there was nothing to look at" are different findings. Emit counters that separate them.
  **Print the zeros**: a census listing only the kinds that fired cannot be told apart
  from one where a kind is dead, so enumerate every expected witness kind with its count
  including `0`, and say which zeros are known limitations of the scanner.
- **An artifact derived from an EXTERNAL input must name that input beside it, and a
  regeneration must be checked against the named input — diff MAGNITUDE is the tell.** A
  comparison against an external oracle was regenerated with the wrong one of two archived
  oracle outputs; the mistake announced itself as a diff in which the ORACLE-side columns
  moved — something the change under test structurally could not do — and at ~20× the
  expected size. Before adjudicating any regeneration diff, ask which columns the change
  could possibly touch; a diff outside that set is an input-identity failure, not a
  finding.
- **A MAXIMAL PRINTABLE RUN IS NOT A STRING — sweep its SUFFIXES, or a string whose
  neighbour ends in a printable byte is invisible.** A C string's END is observable (the
  NUL is what the program itself uses); its START is not — a string simply begins wherever
  the previous datum stopped. So a string sweep built on `[\x20-\x7e]{n,}` silently prefixes
  every name whose preceding constant happens to end printable, and no token split on
  non-alphanumerics recovers it. Measured on one binary: two class names went unrecovered
  for eleven rounds and were written up as *"no string anywhere in the install hashes to
  either"* — a **measured zero that was a defect in the reader**, the same family as the
  immediate-operand under-count above. The donors were ordinary float constants:
  `255.0f` = `00 00 7f 43` ends in `'C'`, `1024.0f` = `00 00 80 44` ends in `'D'`, so
  `CRendScaledFont` and `CSimpleAnimation` were swept as `CCRendScaledFont` and
  `DCSimpleAnimation` — one byte long, hashing to nothing, in four separate DLLs. Yield each
  run's bounded suffixes as well (a 4-byte datum contributes at most 3 printable bytes
  before the run breaks, so a skip bound of 4 covers it). Three follow-ups, all cheap:
  - **The same widening usually feeds a MEASURED ZERO somewhere else, so measure it before
    landing it.** Here the dictionary also carried a "of 588 unresolved ids, exactly one is
    the hash of any string in the install" claim, and a single accidental collision would
    have reopened it. A standalone probe over 6,034,721 runs, run before the edit: the
    skip-0 arm reproduced the old result exactly and the skip-1..4 arm added **zero** ids —
    2.49M → 4.67M distinct strings for the identical zero, i.e. the claim got *stronger*.
    Had the probe come back non-zero, the honest outcome was a recorded cost, not a
    quietly-unwidened sweep.
  - **Require the NUL when you match.** Matching a name as a bare substring is exactly what
    the trap defeats; `name + b"\x00"` is the boundary the distortion cannot fake.
  - **Retract the old claim IN PLACE, and retire the lead it spawned.** The wrong zero had
    grown a follow-up ("their registrars are the best lead, look in this code blob") that
    was never needed — the answer was in a different binary the whole time. A stale lead
    reads as an opportunity forever; strike it where a reader will meet it.
- **A GUARD THAT COMPARES ADDRESSES WHERE IT MEANS VALUES discards whole bodies the moment
  the compiler duplicates a tail.** The conservative rule "the last relevant write in this
  function must be one my tracker attributed, or nothing here is trustworthy" is correct.
  Expressing it as *"last-tracked-write **location** == last-write-of-any-kind
  **location**"* is not. Measured: a compiler duplicated a factory's epilogue across a branch
  instead of joining it, emitting two identical writes of the same value; the register-restore
  instruction between them legitimately kills the tracker, so the second write is seen but
  unattributed — two locations, one value, and the guard threw the entire function away.
  Six exact class sizes were absent from every artifact for months, reported honestly as
  `skipped: no_tracked_construction=6`. The sibling class whose branches merged *before* the
  write had been decided exactly, and sat in the same file.

  Compare the VALUE: every relevant write from the last attributed one onward must carry the
  same value. Three things make that a repair rather than a loosening, and all three are
  cheap:
  - **Assert the direction.** State whether the new rule is strictly weaker or strictly
    stronger and make the probe *count the other direction and raise*. Here a
    `REGRESSIONS (old True → new False)` counter asserted **0**, with an example address in
    the exception. "12 rows changed" alone cannot tell a repair from a hole.
  - **Measure reach over the FIRE POPULATION, not the cases in hand.** The guard had five
    consumers, one of them a mutating applier that the read-only stability harness is blind
    to by construction, so proving the six known cases proved nothing about the rest. Both
    rules were run over every function under every seeding the consumers use — 11248
    body/seeding pairs — and exactly 12 moved.
  - **Point the probe at the PRODUCTION implementation once the rule lands.** Written with
    two local copies of the rule it is a one-off; extracting the rule into the library and
    having the probe call it for both settings turns the same hand-built arms into a standing
    regression test on the shipped code.
- **A SKIP COUNTER is a to-do list, not a footnote — and an absent ROW hides better than a
  blank CELL.** This document already says to print the zeros and to suspect the reader
  before theorising about a blank census. The sharper failure is one rung up: a sweep that
  honestly prints `skipped: no_tracked_construction=6` beside 196 successes, and a decider
  that honestly reports those 6 as "no upper bound". Nothing is hidden and nothing is
  wrong — yet the six classes are absent from the artifact *entirely*, so no census of it
  can see them, and the project's narrative silently became "nothing allocates these"
  (a story about the DATA) when the truth was one line of reader limitation. Measured: all
  six were allocated by an ordinary factory whose `PUSH 0x230` sat twelve bytes before the
  call, and reading ONE of those factories by hand produced six exact class sizes that a
  purpose-built witness round had failed to reach. **Treat every non-zero skip counter as
  a queued item with an owner, and read one skipped case by hand before believing any
  conclusion drawn over the survivors.** A blank cell at least appears in the census; a
  dropped row does not appear anywhere.
- **A CALL TO A BASE CONSTRUCTOR IS NOT AN ALLOCATION OF THE BASE — AND THE MISLABEL PRINTS A
  PLAUSIBLE, PRECISE, WRONG SIZE.** Asking *"is class X ever allocated on its own?"* as *"is there
  an allocation whose object is handed to X's constructor?"* is the natural phrasing and it is
  wrong under single inheritance: a descendant's factory allocates the DESCENDANT and calls the
  base constructor on the same pointer. Measured, on a class bounded [452, 496]: the probe
  answered **2 direct allocations, of 676 and 496 bytes** — which are exactly the two descendants'
  sizes — and printed *"DIRECT ALLOCATIONS of AIUnit -> 676 bytes"*, a confident wrong size for
  the very class the round existed to size. **Classify each allocation against the known
  descendant sizes before attributing it to the base.** Note the direction of the failure: it
  MANUFACTURES a finding rather than hiding one, which is the direction that gets believed.
- **A CLASS'S VTABLE IS NOT ITS POPULATION — IT IS ONLY THE VIRTUAL HALF.** A per-class witness
  built from vtable slot targets silently omits the constructor and every non-virtual member,
  which are compiled as the class and are exactly as good a witness. Measured: a size probe
  scanned 4 slot targets, found nothing above offset 88, and was one step from recording a
  measured zero over a population that **excluded the constructor which had produced the class's
  existing lower bound of 452**. Widening to slots + constructor + namespace members + bodies
  taking a `Class *` this-parameter moved the highest observed access from **88 to 452** — and
  that jump is the check that the widening reached the right code rather than merely more of it.
  Before believing any per-class census, ask whether its population is the CLASS or just its
  vtable.
- **MEASURE A NEGATIVE FROM BOTH DIRECTIONS, THEN RETIRE THE LEAD EXPLICITLY.** "This class's own
  bodies never reach past K" is half an answer; the other half is whether its DESCENDANTS write
  below their own start, which would move the boundary the other way. Both came back zero, which
  turned an earlier round's prose claim — *"nothing observes those 44 bytes"* — into a
  measurement. Then **say the lead is exhausted, name the routes that were run, and name the
  evidence that would still settle it** (here a runtime observation at the descendant's factory,
  recorded at its own provenance tier since it is a fact about one execution). A stale "unknown"
  reads as an opportunity forever, and the next round will otherwise re-derive the same negative.
- **PRICING THE NEXT ROUND IS A REVIEW OF THE LAST ONE — POINT THE PRICING PROBE AT LAST
  ROUND'S ARTIFACT WITH A DIFFERENT QUESTION, BECAUSE IT CATCHES WHAT THE GATES STRUCTURALLY
  CANNOT.** This document already says to price a queued round against the program
  because a plan's premises decay. The other half is that the pricing probe is the first thing to
  look at last round's ARTIFACT with a different question, which makes it the cheapest review you
  will ever get. Measured: a layout round shipped with its tiling guard passing on every class,
  its re-derivation calibration at 198/198 and an outside-route calibration at 500/500 — all
  green, all correct, and all blind. The next round's apply census then printed a stratum named
  *"fields start at 0"* whose members had a recorded base, and **a class with a base cannot own
  byte 0; that is its base's vptr.** The guards were checking internal consistency; only the
  census asked what the rows MEANT. Two things generalise:
  - **A FALLBACK THAT DEGRADES SILENTLY TO ZERO WILL BE READ AS A MEASUREMENT.** The boundary
    computation fell back to "how far did our own walk of the base reach", which is legitimately 0
    for a base outside the walked population — so one value meant both *"this class genuinely
    starts at 0"* and *"we have no idea where its base ends"*. 33 of 99 classes, **79% of the
    field rows**, took the second path while being reported as the first. Make a fallback chain
    record WHICH RUNG it landed on, as a value in the row: here `base_sizeof` /
    `base_ctor_write_max` / `base_observed_extent` / `base_unmeasured`. Same family as
    "distinguish measured-zero from structural-zero", one level down — inside a derivation rather
    than in a census.
  - **WHEN THE ROWS ARE RIGHT AND THE LABEL IS WRONG, ADD THE LABEL; DO NOT DELETE THE ROWS.**
    Excluding the affected classes or blanking their fields would have discarded 2839 correct
    observations — every cell was a real access to a real byte of the object. What was false was
    the claim that they were the class's OWN storage. A `scope` column repaired it, and the
    reclassified rows turned out to be the MOST useful for the apply that follows, because they
    already cover `[0, sizeof)`.
- **A SUMMARY LINE THAT AVERAGES TWO POPULATIONS IS WRONG EVEN WHEN EVERY ROW IN IT IS RIGHT.**
  The same round reported "27603 of 50468 own bytes = 54.7%", arithmetic over a mixture of
  own-storage and whole-object rows. Split, the two real figures are **35.8%** and **58.9%** —
  and the headline sat between them, resembling both and describing neither. Before reporting a
  ratio, ask whether its denominator means ONE thing; this is the reporting analogue of the
  vacuity guard, and it fails in the direction that looks most like success.
- **AND THE SKIP COUNTER CAN BE A FINDING WEARING A FAILURE'S CLOTHES — SPLIT THE PREDICATE
  BEFORE BELIEVING THE COUNT.** The rule above says to treat a non-zero skip counter as a queued
  item. The sharper case is where the counter is not a limitation at all. Measured: a layout
  producer refused 99 classes on `sizeof(base) >= sizeof(class)`, sound reasoning because a base
  at least as large as its derived class contradicts containment. Every one of the 99 was
  **equality, and none was strictly larger** — the ordinary behaviour-only subclass, which
  overrides virtuals and adds no data members. Printed as "99 skipped" that reads as the tool
  failing; the truth was *99 classes proven to add no fields*, established by two independently
  derived sizes (each class's own allocation, and the base's own artifact). One `>=` had
  collapsed a finding and a contradiction into a single bucket, and the finding was the larger
  half. Whenever a guard's condition is a comparison, ask what each side of the boundary means
  separately — this is the "absent row hides better than a blank cell" failure moved one level
  up, into the PREDICATE rather than the reader, where no census of the output can see it.
### Calibrating and pricing a witness

- **CALIBRATE A NEW LAYOUT WITNESS ON THE BASE SUBOBJECTS IT ALREADY WALKS THROUGH — IT IS A FREE
  OUTSIDE ROUTE — AND PARTITION THE RESULT INTO THREE BUCKETS, NOT TWO.** Any construction-chain
  or member-access witness for a derived class necessarily crosses its base subobjects, so
  wherever some *other* machinery has already laid out one of those bases you have a graded
  population for nothing, produced by a route the new witness does not consume. Measured: five
  such bases covered 51 classes, giving **500 cells inside a committed field, 0 straddling a
  committed boundary, 2423 in a committed gap**. Three things make it a real calibration:
  - **The gap bucket must be reported and NOT graded.** A cell landing where the older artifact
    claims nothing is neither agreement nor contradiction; folding it into either side corrupts
    the measure, and it is usually the biggest bucket — here 2423 against 500. It is the
    *payload*: the new witness reaching where the old one did not.
  - **Require agreement, not merely absence of contradiction.** A zero `inside` count must raise.
    Otherwise a witness that is systematically misaligned — every cell landing in gaps — passes
    with a clean sheet.
  - **Put the arm BEFORE the write and poison it on real data.** Shifting one cell two bytes per
    gradeable class fires it (`7 cell(s) straddle`), and because the check sits between the
    harvest and the emit, the poisoned run leaves no artifact to revert. That is the "a run that
    RAISED must not have its output promoted" rule solved by ORDERING instead of by cleanup, and
    it is strictly better because it cannot be forgotten.
- **A REDUCING OPERATION IS WHERE AN EVIDENCE SET GOES TO DIE — READ WHAT YOUR SWEEPS DISCARD,
  NOT ONLY WHAT THEY EMIT.** The skip-counter rule above is about rows a sweep declined to
  produce. This is the case where the sweep *did* the work, on the *whole* population, and
  threw the result away one line later. Measured: a body scanner tracked every store through a
  register its alias tracker resolved to the object, computed `offset + width` for each, and
  kept only the **maximum**, committed as a scalar `ctor_write_max`. 125 of the 133 classes in
  the project's next open lead carried that row — so 125 complete per-offset write sets had been
  measured and discarded, round after round, while a new sweep was being designed to obtain
  exactly them. Two things make this worth a standing check rather than an anecdote:
  - **The reducer's own artifact proves the traversal already reaches the population.** A
    committed `max`, `count` or `any` column is a receipt saying *"this code visited every one of
    these and looked at the quantity you now want"*. Grep your producers for `max(`, `+= 1` and
    early `break` before scoping any round that proposes to go and measure something.
  - **Recovering it is ADDITIVE and must be proven so, not argued so.** Append the cells at the
    exact site that already computes the reduced value, so no existing key's value can move —
    then run the stability harness anyway, because "this cannot have changed anything" is the
    reasoning this document refuses everywhere else. Here: 47 producers, 0 raised, 0 CHANGED and 55
    version-only restamps across 96 artifacts.
- **PRICE A PROPOSED WITNESS IN COVERAGE, NEVER IN SPAN — AND A ONE-EXAMPLE CAVEAT IS NOT A
  MEASUREMENT.** "Measure a proposed mechanism's REACH before building it" has a unit, and
  picking the wrong one flatters the round. Measured on the same population: the committed
  high-water marks reach a median **31.9%** of each class's size, which reads like a workable
  layout; the bytes those same writes actually **cover** are **2.3%**, because one store far out
  produces a large maximum and a tiny footprint. Span is an upper bound on coverage and the gap
  between them is the entire question. The sibling witness failed the same way in the other
  direction: a previous round had called its tiling "sparse" on the strength of one example, and
  over the whole population it covers **2.7%** of bytes with 106 of 133 classes under 5% — the
  difference between "supplement it" and "it cannot carry the round".
- **A CONSTRUCTION WRITE SET IS A PROPERTY OF THE CHAIN, NOT OF THE BODY — FOLLOW THE CALLS
  WHOSE RECEIVER THE SCAN RESOLVED TO THE OBJECT, AND READ THE QUIET RESULT AS THE TELL RATHER
  THAN AS A BROKEN TRACKER.** `references/cpp-abi.md` says a vptr-store chain names ancestors because MSVC
  deletes unobservable stores. The layout consequence is the same mechanism read forward: MSVC
  emits a derived constructor as `call base_ctor; mov [this], own_vftable`, so essentially all
  field initialisation lives in the **base's** body. A flat per-body write set therefore reports
  **one** tracked cell on a 936-byte class, which reads as a broken tracker and is not. Follow
  the calls whose receiver the scan resolved to the object — delta-0 by construction, so sound
  under single inheritance — depth-capped and cycle-guarded: coverage went **5.0% → 69.8%**,
  median **77.6%** per class, 78 of 125 classes above 75%. Two riders:
  - **Another artifact usually already records WHY the witness was quiet.** The size artifact's
    witness column read `ancestor:ctor_write_max` for most of this population: the size was known
    to be inherited, therefore so were the fields. Nobody had put the two side by side. When a
    witness comes back surprisingly quiet, search your own artifacts for the explanation before
    theorising about the tracker or the data.
  - **Say what the coverage number is NOT.** It is bytes touched by an observed construction-time
    access — offsets and widths — not fields identified, typed or named; and the chain admits
    non-constructor callees on purpose, because a write through `this` proves the cell is in the
    object whoever emitted it. Splitting those bytes between a class and its bases is a separate
    step with its own rule.
- **Diminishing returns in one vein is a signal to change AXIS, not to push harder, and the
  trigger is measurable.** When a round costs a full apply-and-verify cycle to decide four
  bytes, and the residue is a handful of items each needing its own bespoke witness, that
  population is exhausted *for that method* — the remaining information is somewhere else.
  Cheap axis changes that repay: rank functions by CALL-SITE FAN-IN rather than by class
  membership (measured: the two highest-fan-in unnamed functions in one binary were the
  game's own allocator and its free, 905 call sites between them, invisible to every
  class-oriented view and already sitting in a script constant, unapplied); ask what a
  thing IS rather than how big it is; and look at the populations no route reaches at all.
  The point is not that breadth beats depth — it is that a stalled depth metric is
  evidence about your METHOD, not about the binary.
- **Before extending a rule to a new evidence source, compare what the RULE requires against
  what the evidence already proved — the extension is often free, and if it is not, that is
  the moment to stop.** Measured: a containment rule (`sizeof(base) <= sizeof(descendant) <=
  alloc(descendant)`) needed only ANCESTRY, while the evidence it was being offered — edges
  from a destructor-chain witness — had already cleared a strictly *higher* bar (immediacy)
  to be recorded at all. So it qualified *a fortiori*: no new soundness argument, no new
  calibration design, one row changed and a class went from an open interval to an exact
  size. The round was short precisely because that comparison was made first instead of the
  argument being re-derived from scratch. The same question asked the other way is the stop
  signal: when the new use needs MORE than the evidence proved, do not extend — that is a new
  witness wearing an old rule's clothes.
  - **Re-run the rule's calibration over the COMBINED population, and demonstrate it firing
    THERE.** A calibration that only ever sees the original evidence passes vacuously on the
    new. Here the poison had to point a *fabricated new-source edge* at a known-size base to
    prove the check reached the added rows at all.
  - **Give the ground-truth arm a NEGATIVE TWIN, and assert its diff is exactly the expected
    size.** A flag that re-runs the decider *without* the new evidence must fail to reach the
    result, and the two runs must differ in exactly the rows claimed. Otherwise "the value is
    now pinned" is a fact about the artifact, not evidence that the new source caused it.
  - **A diff of only provenance cells is the OPPOSITE of churn — say so in the write-up.**
    Two rows here changed `witnesses` and `note` with no value moving: the classes now cite
    their real base rather than their own constructor. A skimming reader sees noise; the next
    reader following the citation sees the difference.
- **A hardcoded population count in a TEST is the same defect as one in a report, and worse
  placed.** A harness baseline asserting "15 hubs" went stale the moment a round legitimately
  added a sixteenth — inside a check that *passes*, so nothing announced its expiry. Derive
  it from the population function the code under test already exposes.
- **Two witnesses that are both LOWER BOUNDS on the same relation must be ANDed, never
  required to AGREE.** The instinct on acquiring a second route to a fact is to gate on the
  two agreeing. That is right for witnesses that can err in both directions and **wrong** for
  the common case where each can only MISS evidence, never invent it — there, a
  symmetric-agreement gate fires on the perfectly normal case of one route being quieter than
  the other. Measured: two routes to "does this class have a subclass" — recorded parentage
  (from constructor vptr stores) and destructor-chain descendancy — were gated on agreement
  and the gate raised on two classes. **The data was right and the gate was wrong**: both
  routes depend on a compiler-emitted store that is *deleted where nothing observes it*, and
  the two get deleted in different functions, so each can wrongly say "no subclass" and
  neither can wrongly say "has one".

  Split the one question into two and the asymmetry becomes usable:
  - **SOUNDNESS** — does the new route ever report a relation the established one
    *contradicts*? This is what licenses it to speak at all, and it is graded only on the
    sub-population the established route can speak about. (Here: 309 of 309 gradeable pairs
    confirmed, 0 contradicted; the 35 pairs touching unclassified entities are *counted*, not
    graded, because there is nothing to grade them against.)
  - **PAYLOAD** — does it ADD a relation the established one missed? For a negative claim
    ("no subclass exists") only an addition can overturn it.

  Then take the **AND**: the negative claim holds only where *every* route is silent. And
  **demonstrate each conjunct separately** — remove a known instance from one route and
  require the other to hold the line, then reverse it. Without those two arms there is no
  evidence the second route was load-bearing rather than decorative.
- **When a rule's CALIBRATION cannot exist in the population you are extending it to, SPLIT
  it and say which half is which — do not manufacture a substitute.** A rule has a *claim*
  and a *precondition*, and they can be calibrated in different places. Measured: the claim
  ("for a subclass-free class, the allocation size is exact") could only be calibrated where
  classes have a size known independently of allocation; the target population has no such
  class and structurally never will. The tempting substitute — grading the rule against the
  members already decided — is **circular**, because those were decided by the very witness
  under test. State the split, calibrate the precondition where you can (it was measurable
  there, and more strongly), and **assert the structural zero** so the split retires itself
  the day the population gains an independent witness.
- **When a round breaks an existing test arm, repair the arm's PURPOSE — never delete it or
  relax its expectation.** Measured: a new pin rule overwrote the value an older arm asserted,
  so the arm stopped testing the route it was written for while still passing a weaker check.
  The fix is to run the old arm with the new rule *disabled* (which is why the new rule needs
  a flag anyway, for the two-step diff) and to add a companion arm stating the **precedence**
  between the two rules explicitly — otherwise the ordering of two `if`s is the only record
  of a decision.
- **A rule built for one population must be offered to its SIBLING populations.** Measured:
  a "leaf class allocates exactly its own size" rule decided 57 classes in one population
  and was never extended to a sibling population with an identical evidence shape (same
  allocation witness, same constructor witness), leaving five leaf classes open whose sizes
  its own arithmetic would have decided immediately. This is the "two consumers of one
  rule" hazard pointed at populations instead of code paths, and it is invisible because
  each population's artifact looks internally consistent. When a rule lands, enumerate
  every population carrying the evidence it consumes and record which ones you did not
  extend it to, and why.
  - **The way to offer it is to MOVE it to one copy — not to copy it, and not to re-derive
    it — and to leave its CALIBRATION where it was demonstrated.** Measured on the second
    occurrence of this hazard in the same project: the sibling tool had grown its own
    narrower version of a base-selection rule, which could not walk through an
    intermediate entity carrying no layout, and so *raised* where the shared rule would
    have walked on. Moving the rule to the shared library cost one insert and two renames;
    the five calibration arms stayed in the original tool's selftest, pointed at the moved
    copy, so the rule relocated and its evidence did not have to be rebuilt. A second copy
    would have doubled the calibration debt and guaranteed the two drift.
- **A PRECONDITION THAT RAISES takes the whole round down with it — where a script is
  DECIDING SCOPE, express it as a printed exclusion; keep the raise only where the state
  already exists.** The mutation-safety rules in this document push everything toward
  raising, and that is right for *damage*. It is wrong for *scoping*, and the two look
  identical in code. Measured: a struct applier raised on one entity that had never been
  named and whose parent had no decided layout — so a bare DRY RUN aborted, one un-nameable
  entity blocked the three that were ready, and the abort landed **before** the round's
  before/after payoff measurement, destroying it. Both conditions became exclusions printed
  with a reason and a count. What makes that a repair rather than a loosening is the
  distinction it draws: for an entity that has **already been applied**, the same conditions
  still raise, because its type exists, so a missing namespace or a vanished prefix source is
  damage. Ask of every precondition: *is this the script choosing what to do, or the program
  telling me something broke?*
- **A WALK added beside a single-step rule must inherit the single-step rule's RAISES.**
  Widening "take the base" into "walk the chain" is the common shape of the fix above, and
  the quiet failure is that the walk *stops* where the single step *raised*. Measured: the
  single-step form raised on more than one recorded base (a single-inheritance
  contradiction); the walk was initially written to stop there and return what it had. That
  is a loosening the two-step diff **cannot** see, because the diff compares outcomes on
  data where the contradiction does not occur. Point the existing poison at both functions.
- **A rule change proven INERT everywhere is indistinguishable from one that does nothing —
  so pin what it DID change with a negative twin.** The two-step diff coming back
  byte-identical is what licenses the change, and it is also exactly what a no-op looks
  like. Add an arm requiring the one case the change was made for to succeed under the new
  rule and to fail under the old one, and derive it from the program with a guard, so the
  round that finally fixes that case retires the arm by name instead of leaving it inert.
- **Running a sibling tool purely as an INERTNESS PROOF is also a CENSUS of that tool, and
  that is where the next orphaned mutator is found.** The stability harness
  (`references/assertions.md`) drives read-only sweeps and structurally cannot run appliers, so an applier's queue of work grows
  silently. Measured: a sibling applier was invoked only to show a moved rule had not changed
  its behaviour, and its dry run reported an entity that had become eligible two rounds
  earlier under a rule added one round after that — an approved-census round nobody had
  noticed. Whenever you touch shared machinery, run *every* consumer's dry run and read the
  population line, not just the pass/fail.
- **PRICE A QUEUED ROUND AGAINST THE PROGRAM BEFORE APPLYING IT — a plan's premises decay,
  and the decay is invisible from inside the plan.** A round described as ready in three
  separate notes had four premises refuted by one read-only census script: two call-site
  counts stale by ~200; one "rename" that was really a MERGE into a namespace an earlier
  round had already created from the same evidence; one that needed no rename at all (the
  class was already named — only the *table* lacked a namespace, which turned a blocked
  struct apply into a one-row artifact edit); and two names colliding with types the
  binary's own demangler had built from export signatures. Every one is a fact about the
  program, and none is reachable by re-reading the notes — which is where all three
  descriptions came from. Two follow-ons:
  - **A name collision is EVIDENCE, not an obstacle.** It usually means an earlier round
    reached the same conclusion by another route. Read what is already there before deciding
    what the round does; the operation is often smaller than planned.
  - **But census the colliding name's TIER.** Measured on the same pair: the pre-existing
    names carried the *analyzer* tier and were absent from the agent-name ledger, because
    they predated the tagging discipline — 22 such names across 7 namespaces. Treating "the
    name already exists" as corroboration would have been self-corroboration with years
    between the two halves.
- **WHEN A RESEMBLANCE ARGUMENT IS ACCUMULATING, FIND THE ARTEFACT THE CANDIDATE WOULD HAVE
  LEFT BEHIND AND CHECK WHETHER IT IS THERE.** Attributing a recovered design to a known
  codebase gets easier the more traits you list, and listing traits is not testing. Measured: a
  game's allocator matched a specific published engine on four behaviours including one unique
  to that engine's later version, and the case was becoming persuasive. Two structural tests
  settled it in minutes — *does the bookkeeping live in the struct that engine uses?* (no: four
  loose globals) and *is the magic value that engine writes past each block actually written?*
  (no: across 113 instructions the only constants stored were `0` and one address). The second
  dissolved the strongest coincidence in the case, because the `+4` everyone had read as that
  engine's fingerprint turned out to compensate the allocator's own round-DOWN two lines
  earlier. **A single absent artefact outweighs any number of shared behaviours** — and note
  which way this failed: the resemblance argument was manufacturing a conclusion, not hiding one.
- **Measure a proposed mechanism's REACH before building it.** A cheap probe that counts
  how many targets a route could possibly reach costs minutes and routinely refutes the
  round you were about to spend a day on. Two measured examples from one session: a plan
  to lay out ~229 classes died when a five-line count showed only 56 were reachable at
  all and 12 of the 16 usable ones were already done; and an interprocedural type
  inference died when its own calibration — the cases where the inference could be
  checked against ground truth — disagreed 18 times in 54. Run the counting probe first;
  a refuted round is a cheap round.

  **And measure reach over the whole population, not over the census you already have** —
  an existing census is often a lower bound *by construction*, in a way its summary line
  does not admit. One here enumerated adjacent pairs of recovered tables, so any instance
  whose tail fell below the noise-filter threshold was dropped before the census could see
  it; scoping the round from its 7 findings would have been scoping from a filtered view.
  The whole-population probe happened to confirm exactly 7 and 0 new, which is the outcome
  that makes the check look unnecessary and is precisely why it has to be run.

### Undefined code and the byte-pattern engine

- **A BYTE PATTERN'S FALSE-POSITIVE RATE IS DIRECTLY COUNTABLE AGAINST THE FUNCTIONS YOU
  ALREADY HAVE — count it before mining anything.** Mining function-start patterns from a
  program's own corpus (`Ghidra/Features/BytePatterns`,
  `ClosedPatternRowObject.mineClosedPatterns`) is the obvious way to attack undefined code, and
  the obvious way to price it — how many candidates would it produce — is the wrong axis.
  **A wrong function start ABSORBS the true function after it**, so the mechanism damages the
  population it is meant to grow, and the function count rises either way. Precision decides the
  round, and your existing functions *are* the graded set: every occurrence of a candidate
  prologue inside a defined body but **not at its entry** is a place the pattern would fire
  wrongly. Measured on one 1999 MSVC/x86 binary with 5,629 known starts: `c3909090`
  (`RET; NOP NOP NOP`) is **54 starts against 3,230 interiors — 1.6% precision**; `c2040090`
  is 29 against 972; only **5 of the top 25** reach 95%. And the vocabulary itself refused the
  round before precision even mattered: **1,950 distinct 4-byte prologues over 5,629 starts,
  top 25 covering 34.1%**. Derive the count two independent ways (per-entry reads, and a bulk
  image scan classified against the same start set) and **raise if they disagree** — that is the
  only thing that catches a misaligned bulk read, which would silently invalidate every number.
- **PRICE THE UNDEFINED-CODE SEARCH SPACE BY CONTENT, NOT BY RANGE COUNT — the filler
  dominates, and the headline number is a span.** `Listing.getUndefinedRanges` over
  `Memory.getExecuteSet()` is the right enumeration and its total is not the population.
  Measured: "5,321 ranges / 87,713 bytes", carried across several rounds, resolved into
  **5,033 ranges / 44,440 bytes of pure `0xCC`/`0x00`/`0x90` alignment padding**, 52 ranges under
  8 bytes, and a candidate-bearing residue of **236 ranges / 43,139 bytes**. 95% of the ranges
  could never hold a function. Classify each range by content before scoping anything on it.
- **`PseudoDisassembler.isValidSubroutine` IS NOT A DISCRIMINATOR — 100% RECALL, 72%
  FALSE-ACCEPT — AND ONLY THE NEGATIVE TWIN SAYS SO.** `ghidra.app.util.PseudoDisassembler` is
  genuinely read-only (its own javadoc: creates no references or symbols, needs no transaction),
  which makes it the right instrument to reach for and the easy one to over-trust. Graded both
  ways on one binary: **400 of 400** sampled known function starts accepted
  (`isValidSubroutine(addr, allowExistingCode=True)`), and **288 of 400** sampled known function
  *interiors* accepted too. So "N undisassembled ranges decode cleanly" is close to
  information-free — a recall-only calibration would have published 224 such ranges as a finding.
  This is the skill's "calibrate by positive agreement, not only absence of contradiction" rule
  pointed at its mirror image: here the recall arm passes trivially and the *precision* arm is
  the one that refuses the instrument.
- **A GAP BOUNDED BY TWO DEFINED FUNCTIONS IS A SAFER PLACE TO CREATE A FUNCTION THAN OPEN
  BYTES — measure the bounding before accepting a stated hazard.** "A false positive absorbs the
  true following function" is the correct objection to pattern-driven function creation in open
  bytes, and it is **void** where the following function already exists in the database.
  Measured: of 236 candidate ranges, **221 were preceded by a defined function ending in a flow
  terminator and 210 were followed immediately by a defined function START** — whole functions
  sitting in bounded gaps of an otherwise contiguous `.text`. The bounding is a safety property
  no byte pattern supplies, and it is one cheap census away.
- **ASK WHETHER AN UNDISASSEMBLED BOUNDARY TRUNCATES A DEFINED FUNCTION — it is cheap and it
  grades your whole evidence base at once.** If a defined body stops at such a boundary while its
  last instruction still **falls through**, that function is cut short in the database, and every
  witness computed by walking function bodies — write sets, construction extents, read/write
  cells, and every layout resting on them — has been reading a partial body and reporting a
  complete answer. Nothing warns. Ask each instruction's own `FlowType` (`isTerminal()`,
  `isJump()`, `hasFallthrough()`), never a mnemonic list, and **calibrate both ways over the
  whole corpus first** — measured, 5,628 of 5,629 defined functions end in a flow terminator and
  1 falls through, which is what makes the answer (**0 of 236 truncated**) a measurement rather
  than a property of the probe. On this binary it was the most valuable thing the round produced,
  and it was a reassurance, not a defect.
- **SEPARATE FEASIBILITY FROM VALUE, AND MEASURE BOTH.** The same population was recoverable
  *and* worthless: **11 references reach all 43,139 residue bytes, every one `DATA`, none a
  `CALL`** — linker-retained code the game never runs, appearing in no tick. A round can be
  perfectly feasible and still not worth running; a pricing probe that reports only reach has
  answered half the question. Say which half you measured.

- **TO ASK WHAT A SHIPPED ANALYZER WOULD DO, BORROW ITS MACHINERY — DO NOT REIMPLEMENT ITS
  RULES.** This is the "check for a built-in before writing your own" rule applied to a *question*
  rather than to a task, and it is the higher-value form. Measured: the question "can Ghidra's
  shipped function-start patterns fire anywhere in these undisassembled bytes" would have required
  hand-rolling ditted-bit matching, pre/post pattern pairing, a fixed-bit budget and four
  post-match prerequisites — four chances to be subtly wrong, each failing in the direction that
  MANUFACTURES a finding. What it actually takes:
  - `ghidra.app.analyzers.Patterns.getPatternDecisionTree()` → `findPatternFiles(program, tree)`
    — **ask which pattern files apply to this program** rather than assuming; it resolves through
    `patternconstraints.xml` and a `ProgramDecisionTree`.
  - `ghidra.util.bytesearch.Pattern.readPatterns(resourceFile, arrayList, patternFactory)` —
    parses them, and `PatternPairSet.createFinalPatterns` applies the bit budget internally.
  - `MemoryBytePatternSearcher(name, patternList)` + `setSearchExecutableOnly(true)` +
    `searchAll(program, monitor)` — the same `BulkPatternSearcher` the analyzer runs.

  **The seam that keeps it read-only is the `MatchAction`, not the search.** The production
  `PatternFactory` is `FunctionStartAnalyzer` *itself*, and instantiating it is NOT read-only —
  its `applyActionToSet` calls `func.setNoReturn(true)` and writes an `AddressSetPropertyMap`.
  Supply your own `PatternFactory` (a plain interface: `getMatchActionByName`,
  `getPostRuleByName`, so `@JImplements` works from PyGhidra) returning `MatchAction`s whose
  `apply` records and whose `restoreXml` keeps the XML attributes — those attributes carry the
  analyzer's extra prerequisites and decide whether a match is unconditional. `DummyMatchAction`
  is the shipped no-op template.
- **A PATTERN FILE IS A CROSS PRODUCT FILTERED BY AN INFORMATION BUDGET, NOT A LIST OF BYTE
  STRINGS.** `PatternPairSet.createFinalPatterns` builds a final pattern from a (prepattern,
  postpattern) pair only if `postcheck >= postBitsOfCheck` **and**
  `precheck + postcheck >= totalBitsOfCheck`, both counting FIXED (non-ditted) bits. Measured, and
  it demolished an apparently perfect counter-example: thirteen NOPs followed by `SUB ESP,0x10`
  matches the shipped `0x90` prepattern against the shipped `0x83ec 0.....00` postpattern — and
  `0x90` supplies 8 fixed bits while that postpattern supplies 19, so at `totalbits="32"` **the
  pair is never built.** Reading the prologue bytes can never reveal this; reading
  `createFinalPatterns` reveals it in one line.
- **A SHIPPED PATTERN CAN BE STRUCTURALLY INCAPABLE OF FIRING WHERE YOU ARE LOOKING, AND ITS HIT
  COUNT STILL READS AS A FINDING.** Post-match attributes are prerequisites, checked in
  `FunctionStartAnalyzer.checkPreRequisites` long after the bytes match: `validcode="function"`
  requires an **existing** function at the address; `after="func|inst|data|ptr|def"` runs
  `checkAfterName`; `validcode="N"` runs `PseudoDisassembler.checkValidSubroutine`. Measured: 628
  of 640 matches inside undisassembled bytes were the `<data>0xcc</data>` `__break` pattern, which
  carries `validcode="function"` — impossible in undefined bytes by definition — and all 628 were
  INT3 filler. Counted naively that is a 640-strong false positive on a census of padding. **Read
  each pattern's attributes before counting its hits**, and note where a refusal is *structural*
  rather than incidental: `checkAfterName`'s fallback `pureDataReferencesOnly` opens with
  `if (!referencesTo.hasNext()) return false;`, so **no references is a refusal, not a pass**.
- **CALIBRATE A BORROWED ENGINE THE SAME WAY YOU CALIBRATE A WITNESS.** A "0 matches in the region
  I care about" result is worth nothing until the engine has been shown to fire where it should.
  The arm: run it over the whole program and require a large share of its matches to land on
  function starts that already exist (measured: 1,205 of 2,905, **41.5%**). Raise if the file
  resolution returns nothing, if 0 final patterns are built, if the search returns 0 matches, or
  if 0 land on a known start — each of those turns the headline zero into a fact about your
  wiring rather than about the program.
### Identity, joins and populations

- **Do not classify a population member by what it is NOT.** "Not a table start, therefore
  interior to a table" is only sound if the tables tile the region — and recovered runs
  almost never tile anything (262 gaps totalling ~37KB here). The wrong label is invisible
  because it is usually right. Compute the actual extents and emit the third bucket
  (`start` / `interior` / `gap`); the gap cases are where the surprises live, since an
  address in a gap may sit inside something the noise filter discarded entirely.
- **A short name is not an identity — join on ADDRESSES, not names.** Demangled short names
  collide across a hierarchy: an override shares its base's name, so two different tables can
  look identical.
  - **And a NAMING ROUND is what proves whether you actually did.** Measured: applying four
    long-established class identities broke **five** separate checks at once, each of which
    had memorised a spelling where it meant an entity — an approved-scope constant recording
    which node a previous round was authorised to add (by its old label), a
    `startswith("UNKNOWN_")` standing in for "is this an unnamed class" (a *tier* question),
    two tests, one of them a **poison that silently went inert**, and two diagnostics. All
    five were repaired by resolving the table ADDRESS through a helper that had already been
    made rename-proof once, for a different class — the fix existed and had simply never
    travelled to the sites that needed it. If your project has never renamed anything, treat
    every name-keyed join as unproven rather than as working.
  - **The dangerous one is a label a producer emits for an entity it CREATES.** A sweep that
    folds in a new node looked its label up in a map built *before* the fold, so the one node
    it adds always fell through to the synthetic `UNKNOWN_0x…` spelling. Harmless while that
    node had no other name; the moment the class was decided, that artifact disagreed with
    every other artifact, a downstream rule joined the two **by name**, and a class silently
    regressed from an exact size to an open interval. Nothing errored.
- **ADJUDICATE A RELABEL BY ASKING WHETHER THE ARTIFACT IS THE OLD ONE WITH THE NAMES
  SUBSTITUTED — a positional diff cannot answer that.** Renaming changes SORT ORDER, so a
  row-by-row comparison reports nearly every row of a re-sorted file as changed and the real
  change hides in the noise. Normalise both sides, substitute the renamed labels, sort, and
  compare as MULTISETS. Measured: 10 of 14 changed artifacts were provably pure relabels,
  two more differed only in the ordering of names inside list-valued cells, and exactly two
  carried the regression worth finding — which had been one line among fourteen.
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

### Samples, headers and new witness kinds

- **A PINNED SAMPLE MEASURES THE ROUND THAT CHOSE IT, FOREVER — AND IT FAILS QUIETLY, BY
  UNDERSTATING.** Measured: an applier scored its own payoff over five literal function addresses
  picked several rounds earlier for a different class. The round that finally mattered applied 13
  fields witnessed in **ten other bodies, none of them in the sample**, so the applier reported
  "no change" and concluded it had bought nothing. The number was true and the claim was false:
  it was a statement about the sample. **When a measurement over a fixed sample returns zero,
  verify the sample intersects the change before reporting the zero** — and prefer a sample
  DERIVED from the same artifact the round is applying, so it grows with the evidence. Here that
  was 5 → 38 bodies and a real payoff of 110 lines. Same shape as a literal row index beside an
  editable header, or a pinned census beside a growing population: **a literal sitting next to
  something that evolves.** Sweep for the shape rather than fixing the instance — in that repo,
  four such lists existed and exactly one was the defect; naming the other three (one already
  derived, two pinned on purpose) is what turns a worry into a closed answer.
- **THE SCRIPT'S OWN WARNING IS THE FINDING, AND IT IS ONE LINE FROM BEING IGNORED.** That
  applier printed *"the payoff did NOT increase ... check the sample reaches bodies these types
  are ON."* Everything needed was on screen; the available failure was to read the clean
  invariants and move on. **An honesty check earns nothing unless the round budgets the
  follow-up** — a printed caveat nobody acts on is worse than absent, because it documents that
  you were told.
- **`csv.DictReader` SILENTLY COLLAPSES A DUPLICATE FIELDNAME AND SHIFTS EVERY COLUMN AFTER IT.**
  Measured: a header carried **10 names over 9 values** (one name twice), so a late column had
  been quietly returning a different field's data — in that case the program version — for
  however long the defect had existed. No exception, no warning, and the values are all
  plausible strings. **Guard the header explicitly**: assert the names are unique and that their
  count equals the first data row's field count. Both are one line, and nothing else in the
  stack will tell you.
- **BEFORE ADDING A WITNESS KIND, NAME THE POPULATION IT ALONE REACHES.** Measured: a producer's
  four existing kinds were all constructor initialisations and serialiser records, so **a field
  written elsewhere and never saved was invisible to every one of them** — and the artifact
  said "no witness reaches these bytes" over a region the game writes seven dwords into every
  tick. The note was true of the kinds it had and false about the program. A fifth kind is worth
  its cost exactly when that sentence can be written for it; if it only agrees with what the
  others already see, it is a confidence upgrade, not a new witness. And **a new kind arrives
  with its exclusion rules already paid for by earlier rounds — do not re-derive them at
  intake**, because a consumer that re-derives an upstream adjudication is a second copy of one
  rule, and the two drift.

### Register conventions you "know" are per-function facts

- **`EBP` IS NOT A FRAME POINTER JUST BECAUSE IT USUALLY IS.** With frame-pointer omission (MSVC
  `/Oy`, on by default at `/O2`) `EBP` is an ordinary general register, and compilers use it to
  hold whatever is hot — including an object pointer. Measured: a probe classifying `rep movsd`
  destinations hardcoded `("ESP", "EBP")` as frame registers, and so read
  `LEA EDI,[EBP + 0x7c]` — the copy that establishes a 268-byte embedded record at
  `<object> + 0x7c`, the single fact the whole analysis line was built on — as *"a write to a
  stack local at offset 124."* **And 124 is 0x7c in decimal**, so the wrong answer looked
  plausible. Decide the frame register **per function**, from the prologue (`PUSH EBP;
  MOV EBP,ESP`), not per program. The same caution applies to any "this register means X"
  assumption: `ESI`/`EDI` as string pointers, `ECX` as `this`, `EBX` as a preserved base — all are
  conventions the optimizer is free to ignore between calls.
- **A SMALL DELTA IN A CENSUS IS NOT EVIDENCE THAT THE DEFECT WAS SMALL.** Fixing the above moved
  **3 of 176** destination classifications — under 2% — and one of the three was the fact the
  entire arc rested on. **Grade a correction by WHICH rows moved, not how many.** The instinct to
  shrug at a 2% change is exactly wrong when the population is heterogeneous in importance.
- **CALIBRATE ON THE PROPERTY THE PROBE EXISTS TO MEASURE, NOT A CORRELATE OF IT.** That probe
  carried a deliberate calibration: re-derive the hand reading that motivated it. It asked for *a
  copy of the right SIZE in the right function* and found seven — and never asked where any of
  them wrote to, which was the entire output of the round that added destination resolution. It
  therefore **passed for two rounds while the one row it was calibrating against was
  misclassified.** A calibration that cannot fail when the headline is wrong is decoration. Pin
  the output, not an input that correlates with it.
- **QUERY YOUR OWN ARTIFACT FOR A FACT YOU ALREADY KNOW BEFORE TRUSTING IT FOR ONE YOU DO NOT.**
  The defect surfaced only because a later task needed the artifact to answer a different
  question, and the row that should obviously have been there was absent. That is a one-line check
  against a known answer, and it caught a two-round defect that every internal consistency check
  had passed.

### An artifact carrying program-state columns must be regenerated AFTER its own round's apply — and if a column flips, fix the COLUMN

**Measured.** A round produced a new artifact, then applied types to the program, then ran the
stability harness — which reported the artifact `CHANGED`. It had: the sweep recorded per-class
program state (`does a type exist`, `which route applies`) and the round's own apply had changed
exactly that.

The tempting fixes are both wrong. Excusing the artifact from the harness removes the only thing
that would notice it going stale later; annotating "expect churn here" is the same thing written
politely. The two fixes that were right:

- **Make the column mean something stable.** `struct_route` was rewritten to name the route a class
  *belongs to*, not whether the work has been done yet — so it reads the same before and after the
  apply. A column that answers "what kind of thing is this" is stable; one that answers "is it
  finished" is a progress indicator and does not belong beside evidence.
- **Make the sweep and the applier share ONE state machine.** They had independent copies of the
  same rule and disagreed about what `applied` meant. Two producers with private copies of one rule
  is the divergence hazard, inside a single repo — and here the applier re-derives the rule and
  **raises if it disagrees with the artifact's own column**, so the two cannot drift apart quietly.

Residual, and worth stating rather than hiding: some columns are irreducibly program-state
(`type_state` here). Those change once, at the apply, and are stable thereafter — so **regenerate
and commit the post-apply version**, and expect exactly one churned generation per mutating round.

### One bracket's raise costs a reach fault too — expect two faults with one cause

**Measured twice, in different rounds.** A probe pins an absolute whole-program invariant (the
datatype count). A round legitimately changes it; the probe raises — the bracket working. But **a
producer that raises does not regenerate its artifact**, so that artifact is then counted
`unchanged` with nothing having re-derived it, and the reach adjudicator reports it as a second,
apparently unrelated fault.

A round that changes the pinned quantity should expect **two complaints and one cause**, and must
not go hunting for a second one. Write the pairing down beside the bracket, because the second
fault names a completely different file and reads like an independent problem.

And when re-pinning: **account for the delta BY NAME, never by subtraction.** Keep a per-round list
of the names the apply is expected to have added and assert the list is present (here 15 of 15),
and have the *applier itself* assert the added name-set across its own mutation — so the count and
the account cannot drift apart on a replay the way a bare number lets them.
