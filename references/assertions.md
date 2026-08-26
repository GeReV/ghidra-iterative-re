# Assertion discipline under iteration

*Reference for the `ghidra-iterative-re` skill. Only `SKILL.md` is loaded when the
skill is invoked; this file is read on demand when its trigger fires. New lessons of
this kind belong here, not in `SKILL.md`.*

**Read this when:** you are writing a check, a gate, a selftest or a poison — or one has
just fired and you are deciding what it means.

**In this file:**

- Poisons, and making a check fire
- Calibration and vacuity guards
- Checks that cannot pass, and checks that cannot fire
- Pricing reach and independence
- What an apply invalidates
- The stability harness
- Counts, literals and approved censuses
- Files, encodings and hosts
- Reading a gate that fired

### Poisons, and making a check fire

- **Prefer checks that already fail.** Where binary ground truth contradicts current
  state, those disagreements are free demonstrations that a check fires. Cheaper and
  stronger than constructed poison.
- **A ranking rule presumes candidates of UNKNOWN mutual order; do not feed it candidates
  the chain already orders.** A nearest-base ranker scored candidates by shared slots and
  flagged a small margin as "ambiguous". Folding every export-declared ancestor of a node
  handed it the parent AND the grandparent, and the grandparent (a strict ancestor of the
  parent, sharing 4 fewer slots) was graded a competitor — five spurious ambiguity flags,
  the most valuable edge among them, each of which would have excluded the node from naming.
  Fold the nearest known ancestor only; the rest is already in the chain.
- **A selftest that derives its poison victim from the artifact inherits the artifact's
  population, and a round that legitimately empties that population breaks the test for the
  right reason.** "Pick any base-less non-anchored node" was a fine derivation until a fold
  left none (`StopIteration`). Derive victims from a property the round cannot empty, or
  guard the derivation and say what emptied it. Related: a poison that "fires" with the
  WRONG exception is not a demonstration — assert the expected message marker, or a
  `KeyError` from a missing table reads as the check working.
- **A poison that edits a row by string replacement must ASSERT it changed something, or it
  goes inert the day a producer fixes the very blank it was replacing.** Measured: a test
  flipped a registration record's `type= ` (blank) to `type=int ` to make an int-vs-float
  check fire; a later round repaired the parser that had left the cell blank, the cell now
  read `type=float`, the replace matched nothing, and the check reported "did not raise" —
  broken by a correct fix in a different round, with no test in that round reading it.
  Compare the row before and after the poison and raise if equal; and when a producer's
  output changes, run every READER's selftest (a stability harness that re-runs producers
  does not exercise the readers).
- **A DRIFT GUARD THAT NAMES THE VALUE IT EXPECTED CATCHES YOUR OWN BUG; ONE THAT COUNTS
  DOES NOT.** Measured: a census probe recovered call arguments off the wrong end of the
  push sequence (cdecl pushes right-to-left, so argument 1 is the push met FIRST walking
  back from the CALL) and reported every site as carrying a non-constant value — i.e. a
  clean, confident census of **zero** items. What refused it was a guard asserting that one
  already-known value was still present, because that guard could state what it wanted and
  what it found. A guard checking only "did we recover anything" passes the empty answer.
  **Assert a known value, not the shape of the answer** — and keep the bug in the probe's
  docstring, because it is the only thing explaining why the guard is written that way.
- **Poison where `expected` and `actual` might share a source** — the classic dead check
  is two reads of the same variable in the same loop compared to each other.
- **Never verify against state you produced this round.** The `SourceType.AI` filter
  enforces this for harvesters; apply it by hand elsewhere.
- **Never verify against an artifact measured wrong**, including its unchecked parts.
- Test verification logic against poisoned *copies of files*, never the live program.
- **Name the QUANTITY a corroboration bar applies to, and check that quantity.**
  "Two independent witnesses" for a field's *type* is not "two witnesses that the
  field is accessed" — and the loose reading passes silently while reading as the
  strict one in every report. Measured: a rule requiring two witnesses counted
  *accesses to a cell* and then took the type from however few of them carried type
  evidence, so a lone pointer-shaped access among 131 decided the cell was a pointer
  "on 90 witnesses". It was an id. Eighteen of twenty applied type upgrades failed
  the corrected bar. Whenever a bar says "N of X", write down what X is.

- **A CONTROL ARM CAN BE A SPECIFIC GROUND-TRUTH FACT INSTEAD OF A HEALTHY-LOOKING
  COUNT — and when it can, it should be.** "The matcher found 1,435 references in a
  region known to be busy" grades only *liveness*: it proves the instrument finds
  something, not that it finds the thing the probe is actually for. Prefer a control
  the probe must **reproduce exactly**, taken from a source the probe cannot
  influence. Measured: a probe whose job was to attribute a memory access to a
  *pointer* was gated on reproducing one instruction read out of the disassembly by
  hand — `MOV dword ptr [ECX + 0x1ec], 0x12`, with `ECX` reloaded from a known
  out-parameter slot four instructions earlier — as an access attributed to that
  pointer. That grades attribution, which is the claim; a reference count does not.
  Give it a poison that removes the control's subject from the population, so the
  gate is demonstrated firing.

- **AN IMPOSSIBLE RESULT IS A DEFECT, NOT A DATUM — ASSERT ON IT, DO NOT PRINT IT.**
  Where two structurally independent bounds exist, check the interval closes the
  right way and **raise** when it does not: a lower bound above an upper bound is
  not a measurement of anything, and printing it invites the reader to split the
  difference or to quote whichever end suits. Measured: a reach probe returned a
  lower bound of 624 for a base class whose derived class allocates 496 in total.
  That single check turned a plausible-looking number into a diagnosis of the
  probe's own contamination. **The best demonstrated-failing poison for such a check
  is the real defect it caught** — disable the correction and the assertion fires on
  live data, which is stronger evidence than any constructed input.

### Calibration and vacuity guards

- **Calibrate a witness by POSITIVE AGREEMENT, not only by absence of contradiction.**
  Require a new evidence source to *reproduce facts already established by other
  machinery* before it may speak about unknowns. A contradiction-only check passes
  a source that is silently misclassifying everything: measured, a classifier
  compared against the wrong vocabulary (`"float"` where the API returns
  `"x87_float"`) sent all 538 float accesses down the integer branch, and only the
  "you must reproduce the known float cells" arm caught it. Absence of disagreement
  is not agreement.
- **A vacuity guard keyed to `checked == 0` can be satisfied by the WRONG population.**
  The guard exists to catch a check with nothing to compare, and it is defeated by any
  non-empty comparison — including one drawn entirely from a *different* class than the one
  under test. Measured: a base-layout check walked the base's ancestor chain, found no rows
  for the base itself, matched 13 belonging to its grandparent, and passed. Guard the
  population you actually mean: if the entity under test is expected to contribute rows,
  require that IT contributed some, and raise naming both sets when it did not.
- **When a check re-derives what a mutator wrote, keep the two derivations SEPARATE.**
  The instinct is to share a helper so the check and the applier cannot disagree. That is
  backwards — sharing makes them agree by construction and verifies nothing. Two
  independent derivations from the same evidence, compared with a raise on disagreement, is
  a cross-check, and the duplication is the feature. Comment it as deliberate, or a later
  tidy-up will "fix" it into vacuity.
- **Prefer a PARTITION to a vote when attributing evidence, and print both buckets.**
  "Three independent registrations agree" is a count, and a count invites the question of
  how many would have been enough. The stronger form is a split of the *whole* population
  with no exceptions on either side: measured, of the 16 fields a base class's descendants
  registered, 13 sat at or above the base's end (so they are the children's own storage)
  and exactly 3 sat below it — all three the same field, in the base's only layout gap.
  Nothing else crossed the line in either direction. Two properties make that far stronger
  than the vote: it is falsifiable by a single counterexample, and **the boundary doing the
  separating was measured by an unrelated witness**, so the clean split independently
  corroborates the size it depends on. A rule that reports only what it matched cannot be
  audited — emit the other bucket and its count too.
- **A verification harness that re-runs PRODUCERS is structurally blind to MUTATORS.** The
  stability harness in this document re-runs every read-only sweep and diffs the artifacts;
  it cannot run the appliers, because running them would mutate. So a round that legitimately
  changes an artifact can orphan an applier's approved census — its hardcoded population, its
  ground-truth poison — and nothing reports it until a human next invokes that applier.
  Measured: a round took a decided-size artifact from 3 pinned entries to 7; the struct
  applier hardcoded those 3 names and gated on `apply 3`, so it refused to run at all
  (correct behaviour, invisible for exactly as long as nobody ran it), and the same change
  silently falsified the assertion in its selftest that used one of the newly-decided
  entries as its example of something to refuse. When a round changes an artifact, enumerate
  **every** consumer and say which tier each sits in; the harness covers the sweeps and
  nothing covers the appliers but you.

  **And the harness is blind a second way: to the producers it EXCUSES.** A stability harness
  earns its keep by re-running producers, so anything it merely *hashes* — for cost, for a
  cross-host dependency, for an append-only ledger — is reported `unchanged` every pass,
  truthfully, because it was never regenerated. Measured: the artifact recording **which types the
  project had applied**, derived by walking the program's own version history, was hashed-only
  because that walk is slow. Nothing re-ran it and no round did so by hand, so it covered
  boundaries up to **v55** while the program stood at **v83** — 23 boundaries missing, including
  the previous round's own type apply, in the very ledger a later round would consult to ask what
  had been applied. **Every excusal transfers ownership from the harness to a human, so it must
  carry a NAMED trigger saying when that human runs it** ("in any round that applies a type, and
  check its last recorded version against the program's"). An excusal with a reason but no trigger
  is how an artifact acquires no owner at all.
- **Give a ground-truth selftest arm a NEGATIVE twin that pins what the rule DEPENDS on.**
  An arm asserting "against the real artifacts this rule recovers exactly X and nothing
  else" proves the rule's output. It does not prove *why* the rule produced it. Add a twin
  that re-runs the same call with the one input the rule is supposed to hinge on moved, and
  require the opposite outcome — here, re-running an ownership fold with the base sized at
  the disputed cell's own offset had to fold nothing, which is what demonstrates the rule is
  bounded by the measured size rather than by the registrants it happens to agree with.
- **A contradiction needs a competing CLAIM, not merely a different string.** An
  opaque block (`uint8_t[N]`) is the *absence* of a claim about its interior, and a
  vector-typed member's components genuinely are floats — so an access disagreeing
  with either is expected, not contradictory. A conflict rule that does not model
  "no claim here" fires on correct data and trains you to widen it.
### Checks that cannot pass, and checks that cannot fire

- **A check can be UNPASSABLE as easily as unfireable — ask whether the population a
  failing gate grades can even CONTAIN the evidence that would satisfy it, because a
  check that cannot pass spends the whole time looking like a real failure.** The unfireable check is the famous one; its mirror is a
  check no run could ever satisfy, which reports a working witness as broken and sends
  you to debug the witness. Measured: a calibration required a pointer-detecting route to
  fire on a cell some *other* route had independently proved is a pointer. In the tool's
  narrow mode the one comparable cell existed, but every access to it came from function
  bodies that mode's own population rule excluded — so the satisfying evidence was
  unreachable by construction, and the same code passed 56-of-121 in wide mode. Two habits
  fall out, both cheap:
  - **Make a failing gate NAME both populations it compared.** "Never overlapped" and
    "disjoint by construction" are the same message and completely different findings;
    printing the two sets turned a day of suspecting the route into one read.
  - **Before believing a gate that says a witness is broken, ask whether the population
    it grades can CONTAIN the satisfying evidence.** If it cannot, the honest outcome is
    to scope the check to where it can speak — and to *assert the exclusion*, so it
    retires itself when the population changes, rather than outliving its reason.
  Do not fix such a check by enlarging the evidence it grades: widening the narrow mode's
  population would have made it pass, and would have changed the population of the route
  that produced an already-applied type.
- **A check with two modes must be demonstrated in BOTH, or the mode you skipped stops
  being reproducible in silence.** The calibration above was added by a round that ran
  only the wide mode, quoted wide numbers in its commit message, and regenerated only the
  wide artifacts. The narrow mode was not re-run for four commits; its committed artifacts
  quietly stopped matching its own code, and the check had *never executed there at all*
  until an unrelated stability harness ran it. Whenever a round touches a code path shared
  by two configurations, run both before committing.
- **A check that lives inside an `if not SELFTEST:` block is not covered by the selftest,
  whatever the selftest's count says.** Three calibrations sat outside the poisonable
  function, so a confident "4/4 demonstrated firing" described the other four and nothing
  warned. Extract a check into a function taking its inputs **and its mode flag** as
  parameters — the flag is what lets one mode demonstrate the other's branch, and in this
  case the narrow mode's real rows demonstrated the wide branch's raise with no poison at
  all.
- **A witness written before an apply must be RE-VALIDATED after it.** The decompiler's
  output is an *input* to most witnesses, and your own struct and type applies change it —
  so a witness can be invalidated by a round that never touched it. Measured: a destructor
  stride extractor deliberately excluded a constant it kept meeting as an *addend*; typing
  an embedded member array turned that same constant into a real `PTRADD` **multiplier**,
  which walked straight past the exclusion and made the measurement ambiguous. Nothing
  failed at the time. It surfaced two rounds later as a gate firing on unrelated work, and
  looked exactly like damage from the round in progress. The exclusion was not defeated by
  new data — it was defeated by the *category* of the data changing underneath it.
- **WHICH ARM OF A SHIPPED HELPER MUTATES IS NOT IN THE JAVADOC, AND AN INHERITED CAVEAT CAN
  NAME THE WRONG ONE — READ THE SOURCE, THEN BRACKET THE READING.** Measured on
  `FillOutStructureHelper`: a handoff warned that `createNewStructure=True` mutates the
  DataTypeManager. It is **backwards** — that arm builds a detached `StructureDataType` and never
  adds it, while the two arms nobody was warned about are the dangerous ones (one rewrites the
  existing struct you meant to cross-check; the other calls `createClass` and `RenameLabelCmd` at
  **`SourceType.USER_DEFINED`**, laundering into the highest tier from inside a helper that reads
  as an analysis convenience). Per-arm mechanics live in `references/api.md`, "Read-only
  witnesses"; the generalisation is what belongs here: **the source read is a hypothesis about
  your install, and a datatype/function-count diff across the run that RAISES is what makes it
  evidence.** Neither half is optional — reading alone gets the arm wrong when the source is
  subtle, and bracketing alone tells you nothing until you have already run it.
### Pricing reach and independence

- **A REACH FIGURE CAN BE STRUCTURALLY MEANINGLESS AND STILL READ AS TOTAL COVERAGE — price the
  mechanism's PRECONDITION, not the availability of a target.** Measured while pricing that
  helper: all **198 of 198** laid-out classes had a recorded constructor site to point it at,
  which reads as complete coverage, and it produced cells for **1 of 12** sampled. The
  precondition is a locatable receiver, and **173 of those 198 recorded sites are not member
  functions at all** — they are the FACTORIES the constructor was inlined into, correctly carrying
  `__cdecl`, in which ECX is not an input, so there is no mapped symbol for `this`. One
  calling-convention census over the whole population costs seconds. Ask "does the precondition
  hold here", never "can I name a target here".
- **BUT PRICE THE PRECONDITION OVER EVERY ENTRY POINT, NOT OVER THE ONE YOUR ARTIFACT HAPPENED TO
  RECORD — THAT IS WHERE THE FACTOR OF TWELVE WAS.** The same round then priced the whole thing at
  **25 of 198** reachable and **12** independently useful, and both figures were wrong, because it
  tried exactly ONE function per class: the address its size artifact stored. The mechanism's real
  requirement is *a member function with a receiver*, and **every non-static member takes `this`**
  — a method touching many fields is a better witness than a constructor setting four. Widening
  from "the recorded constructor" to "any member" took reach to **198 of 198** and the independent
  population from 12 to **151**. Two riders, both measured:
  - **Enumerate a class's whole function population before pricing any per-class witness.** This
    document already warns that *a class's vtable is not its population — it is only the virtual
    half*; here the mirror failed, a witness built from the constructor alone, and the vtable route
    reached **109 of the 173** by itself. Namespace members reached 64, exported `??0` constructor
    manglings **0**. Report the union.
  - **A VTABLE SLOT TARGET IS NOT NECESSARILY THAT CLASS'S METHOD, AND THE CHECK IS FREE.** Slots
    are shared through inheritance — a derived table repeats its base's entries for every method it
    does not override — so a target reached from C may be a BASE's method whose `this` is the base,
    and attributing its field accesses to C is an ownership error. No hierarchy artifact is needed:
    **a target appearing in more than one attributed table is shared by construction.** Measured
    **560 of 1322 shared**, and the arm that mattered: **0 classes reachable only through a shared
    target**. Had that been non-zero, the reach figure would have been an upper bound pending an
    ownership rule.
- **CROSS-TAB REACH AGAINST INDEPENDENCE BEFORE PRICING A CORROBORATION ROUND — the number you
  want is the INTERSECTION, and it is smaller than either.** A second-witness round is worth what
  it can reach *and* speak about non-circularly. A decompiler-derived witness inherits the types
  you applied, so a class you have already typed may hand your own layout back; a class with no
  type cannot. Measured, on the first pass: 25 reachable, 151 untyped, and the cell that decides
  the round — reachable **and** untyped — was **12**. (The reach half was later corrected to 198,
  taking the cell to **151** — see the entry-point rule above. The cross-tab was the right frame;
  one of its inputs was wrong.) Scoping on 198, on 151 or on 25 would each have been
  scoping on a number that does not exist. **And demonstrate the witness on the independent half**:
  the first sample's one working class was typed (persuasive, worth nothing); re-sampling reached
  an untyped class that reproduced **20 of 20** committed field offsets with 0 straddles, and that
  single case is the whole proof-of-concept.
- **A COUNT MATCH BETWEEN TWO PARTITIONS IS A PROMPT TO JOIN THEM, AND THE APPROVAL CENSUS IS THE
  LAST CHANCE TO REFUTE THE ROUND — SO WRITE IT TO BE CAPABLE OF SAYING NO.** Measured, on the very
  next round: those 173 `__cdecl` bodies were written up as *wrong prototypes an analyzer had
  committed*, and a mutating repair round was recommended and approved. The census that was
  supposed to produce its population instead refuted it. The size artifact carried a `route=`
  column — in the same cell the earlier round had parsed the address out of — splitting the same
  198 classes **173 / 25**, identically to the convention split. Joined **per class by address**
  rather than by total: **0 off-diagonal**. They are the same partition. `factory_inline` means
  the constructor was inlined into a factory, so the recorded site is a factory — not a member
  function, no receiver, and **`__cdecl` is correct for it**. Four things generalise:
  - **An identical total on two partitions is evidence of nothing until you join by address.** A
    total can match by coincidence; a ranked decomposition cannot.
  - **Read every column of the row you generalised from.** The refuting evidence was not missing;
    it sat beside the value already being read, and the defect was an unstated assumption (*every
    `ctor_write_max` site is a constructor*) the artifact itself contradicts.
  - **"Not an X" needs positive evidence, not the absence of X's marker.** The confirming arms —
    **173 of 173 return a pointer, 0 of 173 read ECX before writing it**, from raw instruction
    p-code — with the control that makes the zero a measurement: **25 of 25** genuine member
    functions in the same population *do* read ECX first.
  - **A round whose damage no existing gate can see must be refuted BEFORE the apply.** Setting a
    calling convention moves no function count and creates no symbol, so the function-count
    invariant and the AI ledger would both have stayed green while 173 correct functions acquired
    a wrong `AI`-tier prototype under a headline reading *"173 prototypes repaired"*.
- **BEFORE HAND-ROLLING AN ALIAS TRACKER, CHECK `FillOutStructureHelper` — IT IS THE SAME
  WITNESS, ON SSA, AND IT RECURSES INTO CALLS.** A linear register tracker over raw instructions
  is the natural way to answer *"which offsets does this body touch through `this`"*, and it is
  what projects write: it can only LOSE registers, it stops at basic-block joins, and it stops
  dead at a `CALL`. Ghidra ships the same question answered over decompiler SSA —
  `ghidra.app.decompiler.util.FillOutStructureHelper.processStructure(highVar, func, ...)`, then
  **`getStorePcodeOps()` / `getLoadPcodeOps()`**, each a list of `OffsetPcodeOpPair`
  (`getOffset()` / `getPcodeOp()`), plus `getComponentMap()` → `NoisyStructureBuilder`. Because
  `HighFunction` p-code is SSA with `MULTIEQUAL` phis it crosses basic blocks by construction,
  and with a non-null `decomplib` it follows CALLs. Measured on one project: its hand-rolled
  scanner reached a median of 4 construction bodies per class only because the ROUND explicitly
  walked the call chain itself, and its own docstring conceded the tracker "can only lose
  registers".

  Three cautions, and they are why this is a check rather than a recommendation:
  - **It is decompiler-derived**, so it inherits every signature you have applied — the exact
    circularity this document's trust model exists to prevent. It is a second witness, not a
    replacement for an instruction-level one, and the two disagreeing is a finding.
  - **`createNewStructure=True` MUTATES the DataTypeManager.** Use it as an evidence producer:
    take `getStorePcodeOps()` and discard the returned `Structure`.
  - **The returned ops are not confined to the function you passed.** Ghidra's own caller filters
    them (`RecoveredClassHelper.runFillOutStructureHelper` calls
    `removePcodeOpsNotInFunction(function, stores)`), and a harvester that skips that step
    attributes another function's accesses to this one.
- **Scope a candidate pool to the smallest structure the rule is actually about.** That
  same extractor pooled candidates across the whole FUNCTION when the ABI contract it
  encodes (an array walk with its count at `[ptr-4]`) is a property of a **loop**. Pooling
  one level too coarse was correct only for as long as every one of these functions happened
  to contain a single loop — i.e. it was luck, indistinguishable from design, until an apply
  added a second loop.
### What an apply invalidates

- **A decided-artifact RENAME can disable the witness that corroborated it — whenever
  you rename a component, a field or a type, ask which witnesses match on that spelling
  and re-run them in the same round.** The worst version of "re-validate a witness after
  an apply", because it looks like bookkeeping rather than evidence. Measured: a value type's component was renamed
  `w` → `Layer` on the strength of a registration witness that matches *registered
  suffix names against the type's component names*. The rename broke that match, so
  the route silently stopped deciding — and the artifact it had produced stayed in
  the repo, now underivable, for **sixteen program versions**. The apply that
  invalidated the evidence was the one the evidence had justified. Two follow-ups
  measured when that one was repaired, both of which outlive the rename:
  - **The rename was not what hid it — the SILENCE was.** A witness that abstains
    per-item needs a guard separating a *legitimate* abstention from a broken
    route, or the two print the same line forever. Here a genuinely partial
    registration (one component, nothing to decide) and a complete four-component
    one that failed to match were reported identically, which is what bought the
    defect sixteen versions. This is the unfireable-check rule one level down: not
    "the witness produced no rows" but "the witness declined *this* row", and the
    fix is the same shape — enumerate why. Derive the guard's threshold from the
    constants (the known types' component counts), never a literal, or the next
    type added leaves the guard behind.
  - **A correspondence you DECLARED is not a match you measured.** The broken route
    had bridged the two spellings with a hand-written alias; the repaired one
    matches literally, but only because the component now carries the
    registration's own name. Either way the name tests nothing — count, offsets and
    kinds are what discriminate the types. Write that down where the comparison
    lives, or a later reader banks the string equality as a second witness. Same
    family as the disambiguator rule below: the tautology is invisible because
    every report still describes it as agreement.
  - **Repair at the narrowest scope that fixes it.** The tempting fix was to drop
    name matching altogether, making the route rename-proof forever — and it
    decided exactly the same rows, i.e. bought a broader fire population for no
    measured gain. Robustness belongs in the guard, not in a loosened rule; a
    widened rule is a decided-artifact change wearing a bugfix's clothes.
- **A structural fold that replaces N evidence-bearing cells with ONE composite orphans
  every check and row keyed to the old cells — and none of them fails; they go INERT.**
  Folding a 67-cell block into one embedded-record component silently orphaned two
  decided per-cell type upgrades, one must-fire pin, two test poisons, and one
  serialisation cross-check across three consumers. The pattern that caught the tail: a
  raise IN THE MACHINERY for any orphaned key (an upgrade keyed inside a group), pins
  re-scoped with the retirement recorded, poisons rebuilt with assert-it-changed guards,
  and the shared descent extracted on its third consumer. Grep for the old keys before
  the fold, not after the third silent miss.
- **The selective-revert rule binds automation too.** A stability harness classifies
  version restamps against the on-disk working copy; a helper loop that then blanket-
  reverts every restamped file to VCS state wipes the round's own uncommitted regenerated
  artifacts — the exact blanket revert the rule forbids, executed by a script instead of
  a hand. Give the loop an explicit set of the round's own files, by name.
- **THE ANTI-CIRCULARITY AUDIT MUST BE RUN OVER THE WHOLE LEDGER, NOT PER ROUND.** Discharged
  against "the names this round applied", it can only ever catch a leak the round happened to
  disturb — which is why one project found its only such leak in the round that named the
  highest-fan-in function in the binary, and in no earlier round. Run it once over every
  applied name against every cell of every committed artifact. Four things decide whether the
  answer means anything:
  - **Join on the UNQUALIFIED names too.** Measured: only 677 of 3596 applied addresses
    carried a namespace, so a qualified-only audit reaches under a fifth of the population —
    and the one real leak was an unqualified allocator name.
  - **Excuse by (artifact, COLUMN), never by artifact.** A file that legitimately carries an
    applied name in one column would otherwise get a blanket pass for its other fifteen.
  - **A join by name cannot tell YOUR SYMBOL from the BINARY STRING your symbol came from.**
    Measured: ten flagged cells were all a self-announce literal in `.rdata` — the very
    evidence the applied name was derived from, so equality is the expected direction. Ask of
    every match: *would this cell hold this value if we had applied nothing?* A false positive
    on a provenance gate is worse than noise; it gets explained away once and ignored after.
  - **Turn each excusal into a MEASUREMENT the producer can fail.** Where the excused column
    carries an applied name, require the producer's own name column to be SUPPRESSED on the
    same rows; where the artifact's subject is provenance, require it to LABEL those cells as
    yours from its own source column. Both were demonstrated failing on poisoned copies.
  And **print what the audit cannot see, as a number**: a ledger join is blind to a
  project-derived name applied at a HIGHER tier, as are every other ledger-based gate. That is
  not a hole in the audit — it is the reason the tier discipline exists.
### The stability harness

- **Run every read-only sweep after every apply and require byte-identical output.**
  This is the only cheap defence against the whole class above, because it does not
  depend on predicting which witness a given apply perturbs. Build it as a driver
  over the sweeps whose artifacts you commit, and:
  - **print the exclusions with their reasons on every run** — a silent exclusion
    reads as coverage, and the harness's reach has to be a stated number;
  - **classify benign churn separately.** Most sweeps stamp rows with the program
    version they harvested at, so a re-run rewrites that column and nothing else. On
    one project 14 undifferentiated "CHANGED" files resolved into **19 restamps and 7
    real changes**; a gate that noisy stops being read. Classify, list, never drop;
  - **do not auto-restore what it overwrites.** Sweeps write before their own final
    checks, so a differing run leaves plausible files on disk. Reverting them is a
    deliberate act (`git checkout`), not a side effect that hides the finding;
  - **and when you do revert, revert SELECTIVELY.** The driver regenerates every
    artifact, so running it to *verify a repair* leaves the repaired files sitting
    next to two dozen incidental restamps — and the blanket `git checkout --` the
    gate's own message suggests will wipe the round's work along with the noise.
    Keep the round's files by name, revert the rest.

  Run it *today*, not only after the next apply: any sweep whose current output
  differs from its committed artifact is a perturbation that already happened and
  nobody noticed. Read the artifact's own `program_version` before predicting what
  a re-derivation should produce: "it will return to the committed value" is only
  valid if the committed file was stamped at the *current* version, and a
  prediction that quietly assumes otherwise fails for reasons that look exactly
  like damage from the round in progress.
- **A disambiguator for a witness may NOT consult the quantity that witness corroborates.**
  When two candidates survive, picking the one that matches the already-decided value is
  trivially available and usually right — and it silently converts an independent
  measurement into a tautology, while every report still describes it as two agreeing
  witnesses. Disambiguate on structure the other witness knows nothing about, or report
  nothing.
- **An approved census is a RULE plus a number — record the predicate beside the count,
  or the approval is unreproducible.** The number a human approves is the output of a
  measuring rule, and the rule is the part a later implementer actually needs: measured,
  a handoff recorded "44 approved" and the operation to perform, but the census's
  concept-join bar lived only in a design exploration's chat report — the re-implementation
  derived the population the way the existing machinery always had and fired **100**. The
  population-drift gate (raise if the fired count differs from the approved one) turned
  that into a refused dry run rather than a wrong apply; the bar was then recovered from
  the archived exploration report. Two halves, both cheap: commit the predicate (or the
  census artifact itself) in the same change that records the approval, and when a lost
  rule must be reconstructed, require the reconstruction to reproduce the approved number
  per stratum, not merely in total — a total can match by coincidence; a ranked
  decomposition cannot.
### Counts, literals and approved censuses

- **An expected COUNT must be derived from the artifact, never written as a literal.**
  A hardcoded population size is correct on the day it is typed and silently wrong
  forever after, and because it usually lives in a *summary line* rather than an
  assertion, nothing fails — the round just reports a denominator from an earlier era
  of the project. Two instances here: a test asserting "3 upgrades applied" that went
  stale when a fourth route added rows, and a sweep reporting "198 of **289**
  unanchored tables" that kept saying 289 after 7 of those tables were merged out of
  existence. Compute it (`len(tables) - len(anchors)`), and treat a literal in a
  report as the same defect as a literal in a check.
  - **The worst place for such a literal is a GATE'S DOCUMENTED EXPECTED OUTPUT, and the
    round that invalidates it will fix only the clause it came for.** A checklist recording
    what each gate must print when it passes is prose: nothing executes it, so nothing can
    fail. Measured: a round moved 22 names into a provenance ledger, which changed **six**
    numbers in one audit's documented headline — and that round corrected exactly **one**,
    the number it was *about*, leaving five stale in the same paragraph, among them a
    closing sentence still asserting the very quantity the round had reduced to zero. They
    survived until an unrelated round happened to run the gate and read its real output.
    Wrong code fails loudly; a wrong expected value teaches the next reader a false number
    and is then believed. **When a round changes what a gate PRINTS, re-derive that gate's
    entire documented paragraph from a fresh run — not the sentence you noticed.**
- **When a gate fires during a round, ISOLATE before attributing it to your change.**
  The reflex is to assume the work in progress broke it, and that reflex is expensive
  in both directions: it can send you rewriting something correct, or — worse — get
  the failure quietly written off as "expected fallout from this round" and dropped.
  Re-run the same check against the **pre-change state**. Here a size sweep raised on
  a class's ambiguous destructor stride in the middle of an unrelated amendment;
  re-running it against the pre-round artifact produced byte-identical output
  including the same raise, which exonerated the round in one step and converted the
  failure into a separate, honestly-recorded open item. The control run is usually
  minutes, and it is the difference between a finding and a guess.
- **Diff against a state the failure could not have produced.** The regeneration diff is
  the workhorse verification in this document, and it has a blind spot: if you revert an
  artifact and then re-run the producer, "unchanged" is *also* what you get when the
  producer crashes and writes nothing. Pass state and failure state become identical —
  the unfireable-check shape wearing a workflow's clothes. Measured, and published as a
  false green before it was caught: a sweep died on a `TypeError` in a newly added gate,
  wrote nothing, and the byte comparison against the just-restored file reported a perfect
  match. Two habits close it, and you want both: **read the run's exit status before you
  read its output** (with an async job runner this is not optional — a task id is not a
  result), and **assert the write actually happened** — row count, mtime, a line the
  producer only prints on success — before trusting any comparison.
### Files, encodings and hosts

- **Never byte-compare a tool-written file against your VCS's stored copy.** Ghidra writes
  CRLF on Windows and git normalizes to LF on commit, so `git show HEAD:file` versus the
  working file differs by exactly one byte per line — on every file, regardless of
  content. It looks like a real diff and it is pure line-ending policy. Compare parsed
  content, or normalize first; a harness that compares on-disk before/after is immune and
  is the better design for this reason alone.
- **A WRITER THAT DOES NOT NAME ITS ENCODING PRODUCES DIFFERENT BYTES DEPENDING ON WHICH HOST
  RAN IT — AND THE FAILURE LANDS IN YOUR GUARD RAIL, NOT IN AN ARTIFACT.** The CRLF rule above
  is the famous half of this; the encoding half is quieter and worse. Measured: a shared
  `append_csv` helper opened its file with no `encoding=`, so it wrote **cp1252 when called from
  Windows-hosted Ghidra and UTF-8 when called from a Linux-side script**. A `§` in one round's
  provenance strings became a bare `0xA7`, and a standing Linux-side audit died with
  `UnicodeDecodeError` **reading the very ledger it exists to check** — so the symptom was "the
  gate is broken", not "the data is wrong", which is the reading that wastes an hour. What
  proved it was the helper rather than the round: **four artifacts already carried the same
  character as `0xC2 0xA7`, valid UTF-8**, written by the other host. Name the encoding in
  shared writers; until then, keep everything an applier writes to a ledger ASCII. Any project
  driving Windows-hosted Ghidra from WSL or a container has both halves of this.
- **A LOWER BOUND ON A CLASS SIZE IS INHERITED; A CONSTRUCTOR'S OWN WRITE EXTENT IS ONLY THE
  ANSWER FOR A ROOT.** Obvious stated plainly, and easy to get wrong in code because the witness
  is per-body. Measured: a derived class whose constructor writes nothing but the vptr and a
  class-identity tag has an own `write_max` of **8**, while it cannot be smaller than the base it
  contains — whose constructor reaches **452**. A producer that took each class's own writes
  reported `8` and thereby contradicted a finding the project had already written down in prose.
  Walk the chain (`max(own, base's lower)`), take the UPPER bound from the smallest **transitive**
  descendant that is actually allocated rather than from direct children only, and assert
  containment both ways so a single row can falsify it. And the thing that caught it: **when a
  new artifact replaces prose, diff it against the prose** — the disagreement is the check.
- **A GUARD WRITTEN AGAINST SELF-HARVEST CAN REFUTE THE EVIDENCE RULE ITSELF, NOT JUST THE
  PLUMBING — SO READ WHAT IT SAYS BEFORE RELAXING IT.** A producer required every class name it
  emitted to be attested in the binary's export manglings, on the narrow grounds that a recent
  naming round had applied those same strings to those same addresses. It refused to run — and
  the refusal was not about a read-back at all: it showed that the round which first attributed
  those classes had leaned on *"the factory's NAME declares what it builds"*, while the factory
  manglings actually name their **input** (`CreateFriendlyUnit` takes a `CBasicUnit *`;
  `CreateProject` names the *unit* type, not the project type). The attribution had been resting
  on the English of a member name the whole time. **Emit the NAME and the ATTRIBUTION as separate
  columns at separate tiers**, cite the mangling that genuinely declares the type, and let the
  guard check only the half it can verify.
- **FIXING THE WRITE SIDE OF A ROUND-TRIP WITHOUT THE READ SIDE TURNS AN ACCIDENTAL PASS INTO
  REAL CORRUPTION.** The encoding rule above tells you to name the encoding; this is the trap
  waiting when you do. A bare read and a bare write on the SAME host cancel out — cp1252 decodes
  `0xC2 0xA7` to `Â§` and re-encodes it back — so a file round-trips byte-identically **by
  accident**, and nothing has ever looked wrong. Name the encoding on the WRITE only, and that
  `Â§` is emitted as `0xC3 0x82 0xC2 0xA7`: a file that was correct is now double-encoded, *by
  the change made to fix encoding*. Measured on the first producer run after a writer-only fix
  (non-ASCII `[167,194]` → `[130,167,194,195]`), caught because the check counted the specific
  bytes rather than diffing. **Trace the whole path a value travels — which reader put it in
  memory, which writer puts it back — and change every hop or none.** And note where the
  evidence came from: a byte census, because the naive `git show HEAD:file` diff was drowned in
  CRLF-vs-LF noise, exactly as this document warns two rules up.
- **WRITE THE ENVIRONMENT PROBE TO RUN UNDER BOTH INTERPRETERS, AND RUN IT IN BOTH PLACES
  BEFORE BUILDING ON A ONE-SAMPLE INFERENCE.** A cross-host project has two Pythons with
  different defaults, and a single observed byte is enough to *deduce* which — but not enough to
  find the other direction. Measured: the deduction "Ghidra writes cp1252" was correct, and the
  probe that confirmed it also found the half nobody had looked for, that the reverse direction
  is **silent** rather than raising. Make the probe touch no host-specific API so the comparison
  is one command in each place; it costs minutes and it is the difference between fixing a
  one-directional bug and a two-directional one.
### Reading a gate that fired

- **APPLYING NAMES TO A POPULATION A HARVESTER READS IS THE CHEAPEST AUDIT OF THAT HARVESTER'S
  FILTER — AND THE PASSING DIRECTION IS THE ONE WORTH CITING.** The trust-model reference
  (`references/trust-and-circularity.md`) already says a wide naming round finds filter
  holes, because that is how the only leak on one project was found. The mirror is worth stating separately, because it looks like churn and gets reverted:
  measured, naming 16 vtable slot targets made the project's OLDEST sweep — the one that had
  once folded 578 agent-applied names back into committed evidence — move 12 `primary_name`
  cells from the default `FUN_…` spelling to **blank**. It met sixteen freshly applied names and
  emitted none of them. **That artifact diff is the anti-circularity filter demonstrating itself
  on live data; keep it and cite it rather than reverting it as noise**, because otherwise the
  filter goes unwatched from the round that installed it onward. It also tells you the two
  producers disagree about how to render "no non-agent name" (one blanks, one falls back to the
  default spelling) — harmless, and worth knowing before you join on that column.
- **A verification pass is not a checkpoint — a harness must not pollute the evidence it
  grades.** One sweep here appended ten metrics rows every time it ran, so each pass of
  the stability driver wrote ten rows of non-history (hundreds accumulated, all identical
  in shape). Snapshot-and-restore the append-only files, and *print* that you did, so the
  restore is visible rather than a silent side effect.
- **A FALSE POSITIVE on a gate is worse than noise.** Noise gets skimmed; a false positive
  gets *reasoned about*, and then discounted — which trains the reader to discount exactly
  the entries the gate exists to raise. Measured: a stability harness classified benign
  version restamps by splitting CSV rows on `,` instead of parsing them, so any row with a
  comma inside a quoted field kept its version cell and its file was reported as a
  perturbed witness. It fired on three artifacts immediately after a large type apply —
  indistinguishable from the exact damage the harness exists to catch — and all three were
  pure restamps. **Parse the format; never split it.** A classifier that separates benign
  churn from real change is load-bearing, so it deserves the same rigor as the check it
  feeds.

  **And the worst false positive of all sits on a DISAGREEMENT report.** Measured: a probe
  comparing a property registrar's field names against the names already on the recovered
  structs reported **18 CONFLICTs** — "two sources disagree about this field" — and every one
  was an agreement seen at the wrong granularity. The registrar names `Pos.X`, `Pos.Y`,
  `Pos.Z` at three consecutive offsets; the program has ONE 16-byte component called `Pos`
  covering all three. A string comparison calls that a conflict. **Compare names at the
  granularity each source actually speaks in** — a dotted name is a sub-field of a record,
  not a competing claim about the same cell — and note the asymmetry that earns this its own
  rule: ordinary noise gets skimmed, but a conflict gets *reasoned about* and then explained
  away, which is how a reader learns to discount the conflicts that are real.
- **A confinement predicate cannot see an INTERPROCEDURAL dependency, and a stray is not
  automatically damage.** The usual predicate — signature ∪ local-variable types ∪
  referenced typed globals — misses a function that merely *calls* something returning a
  typed pointer and then dispatches through it: the typed locals in its decompilation are
  decompiler-invented, not committed variables, so no route sees them. Measured: the single
  control stray after a vftable retype was a real, intended effect arriving exactly that
  way. Adjudicate a stray by **reading the function**, and record the predicate's blind
  spot — widening the control group to make the number look better is backwards, and is the
  temptation this rule exists to name.
- **A run that RAISED must not have its output promoted.** Scripts commonly write
  their artifacts before the final verification stage, so a failing run can leave
  perfectly plausible files on disk — already regenerated, already looking current,
  and now disagreeing with the last verified state for reasons nobody recorded. After
  any raise, diff the artifacts and revert the ones that run touched. This is the
  same shape as the apply whose post-write check crashed: the write survives the
  failure, so the failure has to be treated as reaching the *files*, not just the
  exit status.
- **THE SAME VALUE CAN COME BACK BY A DIFFERENT AND SOUND ROUTE, AND A VALUE-BASED ASSERTION
  CANNOT TELL.** Measured: an arm asserted that a particular field offset must NOT appear in the
  harvested evidence, because its earlier appearance had been an artifact of a stack-tracking bug
  since fixed. The offset returned, and the arm failed for several rounds. Adjudicated, it was
  **real**: supplied by a function whose own export mangling declares the type of the parameter
  being read, from the same body that supplies every other offset the check requires to be
  PRESENT. The arm had conflated *"this value must not appear"* with *"this value must not appear
  FOR THAT REASON"*, and only the second was ever the point. **Assert the PROVENANCE, not the
  number** — the repair replaced one value check with three that name the contributing body and
  instruction, plus a negative twin scoped to the artifact's original route, which still fails if
  the bug returns while the value is legitimately present. Note also which way this failed: the
  stale check made a correct harvester look broken, and the tempting "fix" was to go and debug
  the harvester.
- **A PATTERN THAT WAS A BUG LAST ROUND IS NOT EVIDENCE OF A BUG THIS ROUND.** The failing
  instruction above was `TEST byte ptr [EBX],0x8` — reading operand 0, which the immediately
  preceding round had just proved was being mistaken for a WRITE elsewhere in the same codebase.
  It is not the same bug: that consumer scans EVERY operand and names its rows `..._access`,
  which is direction-neutral by construction and correct, because a read of `ptr+K` proves the
  cell lies inside the pointee exactly as a write does. **The identical instruction shape was a
  defect in one consumer and correct behaviour in another.** Adjudicate each consumer against
  what it CLAIMS, not against the shape that burned you last time — pattern-matching a fresh
  finding onto a recent one is how a correct component gets "repaired".
- **A PARTING SPECULATION IN A COMMITTED ROUND RECORD IS INHERITED AS A FINDING — TEST IT, THEN
  STRIKE IT WHERE A READER MEETS IT.** That preceding round closed by predicting the same defect
  in three more consumers. All three were wrong: two scan every operand, one reads the source
  operand deliberately, and the affected consumer was the only one. Notes are read as evidence by
  whoever comes next, so the refutation was written back into the original section rather than
  left to a later reader to discover — the same discipline as retracting a measured zero in place.
- **A check that fires may be a defect in the CHECK.** Validating a decided artifact
  with a *stronger* predicate than the calibrated rule that produced it rejects rows
  the producing sweep legitimately attributed. Before loosening a rule or editing
  data, ask which predicate the artifact was derived under — this is the mirror of
  "a calibration population must share the fire population's code shape."

---

### A tool property your design rests on is a standing gate, not a one-off check

**Measured.** A round represented 19 derived classes as `struct Derived { Base base; }` — a *typed*
member rather than a flattened copy — specifically so they would track their base and could not
drift. The whole argument rests on one property: **that Ghidra propagates a length change through a
typed member.** That is a property of *the tool*, not of the binary, so a version upgrade could
remove it and nothing else in the project would notice: the structs would still exist, still have
the right length today, and silently stop tracking.

So it is measured, and the measurement is **driven on every canary pass**, not once at design time.

Two things make it a real check rather than a ceremony:

- **Run it where it cannot touch the program.** A `StandAloneDataTypeManager` is a real
  `DataTypeManager` with no program behind it, so throwaway types can be built, grown and read back
  with zero blast radius. Bracket it anyway — assert the *program's* type count is unmoved either
  side, because "this cannot have touched anything" is the reasoning this discipline refuses
  everywhere else.
- **Give it a negative control, or it proves nothing.** Alongside the typed member, build the
  design you are *rejecting* — here an opaque `byte[N]` copy of the same base — and require that it
  does **not** follow. A run where both appear to track is measuring something other than
  propagation, and without the control it reads as a pass. The control also produced a finding of
  its own: the project's 13 existing opaque-copy structs are drift-**invisible**, not drift-free.

**Generalise:** whenever a design's safety argument reduces to "the tool does X", write the
smallest experiment that would fail if the tool stopped doing X, pair it with a control that must
fail, and put it in the recurring gate set. Design premises decay silently; program facts get
re-derived every round.

---

## Checking a delegated batch: the three checks that are not about the evidence

When work is delegated in a fixed schema — a batch of proposed names, types or layouts, each with
citations a probe can re-verify — the obvious check is *do the citations hold against the program?*
That check is necessary and it is not close to sufficient. Measured on a 104-row batch where **every
citation in every row it flagged verified perfectly**:

**1. Pre-flight, before the apply, not after.** The citation verifier normally runs over committed
data and therefore requires the rows to exist in the ledger. Give it a mode that runs the same
checkers over an **uncommitted candidate file**, with the ledger join skipped **by name** in the
output rather than silently dropped. It refused ten fabricated citations in the first batch it saw.
Found after the apply, each one costs a program version and a rollback; found before, nothing.

> A verifier that can only run on committed data finds everything exactly one round too late.

**2. The collisions no per-row check can see.** Three defect classes are invisible from inside any
single row *and* invisible to a citation checker:

| what | example |
|---|---|
| the proposed name is **already applied at a different address** | `LoadScreen::LoadScreen` proposed for one address when the ledger has carried it at another for 56 program versions |
| the same name proposed for **two addresses within the batch** | two batches independently reached the same label for different bodies |
| a row with **no name at all** | a reader that declines should emit no row; one that emits a blank leaves the applier to invent something |

Each needs a join over *the whole batch plus the whole ledger*. On the batch that motivated this, a
careful human read-through caught the first three defects in the delegation and **would have missed
the collision**, because nothing about that row looks wrong in isolation. That is the argument for
making the adjudicator a committed script with a selftest rather than a review pass.

**3. Refusal rules need their refusals sampled — a rule that refuses real evidence looks exactly
like a rule working.** The adjudicator's "is this class real?" rule reported a class as invented. It
had no exported member and appeared in no namespace anywhere — but four exported **mangled
signatures returned a pointer to it**, which is the linker asserting the class exists and spelling
its name. A drop is silent by construction: nothing downstream ever says *"a row was refused for a
bad reason."*

> Every refusal rule needs a selftest arm for **what it must NOT refuse**, written at the same time
> as the arm for what it must. Poison the rule and confirm the count moves in both directions.

**A delegated-batch adjudicator's rule set, as a starting point** (each demonstrated firing by
poisoning it, both directions):

- citations whose address is outside the image — external-space ordinals look like addresses
- a namespace/class the project cannot already justify from ground truth
- a qualified name already applied elsewhere, or proposed twice in the batch
- a citation that names *your own* applied markup rather than ground truth — the anti-circularity
  rule, at the adjudication layer, where it costs nothing
- two citations that are the same fact twice (an import and that import's own IAT slot)
- the confidence bar re-applied *after* citations have been dropped, demoting rather than deleting
- blank names and duplicate addresses

---

## Two things that bite when evidence is graded, not just checked

### Certainty and citability diverge, and the tier is where that goes

The most obvious identification in a batch is frequently the one with the **least** citable
evidence. Measured: a four-byte accessor, `return *(this+0x1c)`, whose reading is not seriously
arguable — and which touches no global, calls nothing, references no string, so exactly **one**
witness kind could be produced for it. Under a two-witness bar it is recorded at the lower tier,
beside readings far less certain than itself.

That is the grading working, not failing. The temptation at that moment is to reach for a second
citation that *technically* verifies — a neighbouring global, the vtable it happens to sit in — and
that is the move that turns a tier into a rubber stamp.

> **Record the reading at the tier its evidence supports, and say in the row why the evidence is
> thin.** A confident reading with one witness is more useful to the next round than a padded one
> with two, because the padding is invisible six months later and the note is not.

The corollary for anyone setting a bar: expect small leaf functions to cluster at the bottom tier,
and do not read that clustering as low quality.

### A checker that resolves a name through an artifact cannot see what the artifact omits

A citation of the form `slot <k> of class <C>` had its checker resolve `<C>` to a table address
through a committed hierarchy artifact. Classes absent from that artifact — which, on this project,
was *by definition* the entire population of unattributed tables — were **unciteable, even though
the slot pointer was sitting in memory the whole time.** The check was not wrong; it simply could
not be reached.

Letting the citation name the **address** (`slot <k> of table 0x…`) removed the lookup, and with it
the failure mode. Same rule as every other join in this document, applied one level up:

> **When a check needs a lookup to reach the program, the lookup is the weak link.** Prefer a
> citation that names a program address over one that names a label some artifact must translate.

Two consequences worth planning for: a checker's coverage is bounded by the artifacts it consults,
so **enumerate what those artifacts exclude** before believing a population is unciteable; and when
you add the address form, keep the name form working — existing rows rest on it, and loosening is
safe where replacing is not.

## A gate keyed on IDENTITY cannot police a change to an ATTRIBUTE

A ledger gate that joins "every markup symbol" to "every ledger row" **by address**, in both
directions, is the standard whole-program consistency check and it is a good one. It catches an
off-the-books apply (a symbol with no row), a rollback (a row with no symbol), and any drift in the
counts.

**It is structurally blind to a RENAME.** Change the name on one side only and the address is still
present in both directions, both counts are unchanged, and the gate passes. Measured: a project ran
this gate for dozens of rounds and it was never wrong, because every applier it had ever seen
**appended** — the failure mode simply could not arise. The first applier that rewrote a row
introduced a defect class no existing gate could express.

- **When you add an operation whose failure mode no existing gate can state, name the gate that does
  cover it — in the new script's own header**, where the next person looks. Here that was a separate
  probe joining by NAME rather than by address.
- **Ask of every gate: what does its KEY make invisible?** A join on address cannot see a name; a
  join on name cannot see an address move; a count sees neither. This is the same failure as *a
  wrong join key produces a MISSING answer, not a wrong one*, pointed at the gate rather than the
  harvester.

## Re-tighten a ratchet when the backlog SHRINKS

A ratchet — a committed ceiling on some backlog, asserted every gate pass — exists so the backlog
cannot grow without somebody deciding it should. Raising it is a deliberate act, usually recorded
with the round that earned it.

**Lowering it is just as load-bearing and is the half that gets forgotten.** A ceiling of 73 against
a true value of 68 silently permits five new unjustified entries, and the gate will report healthy
the whole time. The ratchet only measures anything while it is touching the number it tracks, so
move it down whenever the work moves the number down.
