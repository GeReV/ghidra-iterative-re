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

## A ratchet is an instrument only while it is FLUSH against the true value

A ratchet — a committed ceiling on some backlog, asserted every gate pass — is usually thought of
as a guard against growth, so the instinct is that only RAISING it matters. That is backwards, and
the cost is invisible.

**A ceiling left above the true value grants exactly as much unnoticed growth as the slack.** With a
ceiling of 73 against a true value of 68, five entries can accumulate one round at a time while the
gate reports healthy at every step.

**Measured.** One round promoted five backlog rows out of the tier and then tightened the ceiling
73 → 68 to follow — a step that read as bookkeeping at the time. Six rounds later another round
added **exactly one** row, and the gate raised at **69 vs 68**, forcing that single row to be
justified in writing before the ceiling moved. At 73 it would have been silent.

- **Lower the ratchet whenever the work lowers the number.** Slack accumulates across rounds because
  nothing reports it.
- **Price the tightening by what it will make VISIBLE, not by what it finds.** Its value never shows
  up in the round that does it; it shows up in the next round that adds one.
- **A ratchet that has never fired is not evidence of discipline — it may simply be slack.** Check
  the distance between the ceiling and the current value. If that distance is not zero, the gate is
  measuring nothing until the backlog grows past the gap.

## A poison that cannot REACH its guard is not a poison

A population guard was written inside a per-class loop, *after* the filter that selects the classes
to repair. It could therefore only ever fire on a class that had already passed classification — so
a misattributed class that classifies as *sparse*, which is most of them, could never trip it. The
check sat in the file reading like a defence, and was unreachable.

Moving it to run over the whole population **before** anything is classified made it reachable, and
then it needed **two** poisons rather than one: any injected class present in both parentage records
trips the earlier *contradiction* arm first, so demonstrating the *descendant* arm required a class
one record has never heard of. **If one poison fires an arm you were not aiming at, you have not yet
demonstrated the arm you were aiming at** — add the second poison rather than accept the first
exit code.

## A poison that reports "did not fire" is a claim about the HARNESS first

The previous item is a poison that could not reach its guard because of where the guard sat. This one
is the same failure caused by the *test harness*, and it is more dangerous, because its output invites
you to go and "fix" a check that was working perfectly.

**Measured.** A guard inside a generator's merge step had just had its condition widened, so it was
re-poisoned: copy the input directory, corrupt the relevant file, run the generator against the copy,
expect a raise. It reported **"DID NOT FIRE — the guard is inert."** The guard was fine. The generator's
`main()` ignores `argv` and reads module-level constants for its input and output paths, so the
poisoned directory was never opened; the run read the real, healthy artifact and passed, exactly as it
should have.

Two rules:

- **Before believing a poison's verdict — in either direction — prove the poisoned input is what the
  code under test actually read.** A raise proves reachability by itself; a *pass* proves nothing until
  you can show the input arrived. Print the value the code loaded, or assert on a census the poison
  should have moved.
- **Prefer calling the function directly with the poisoned argument to influencing it across a process
  boundary.** Every layer between your poison and the guard — argv parsing, a module constant, an
  environment variable, a config file, a cached path — is a place the poison can be silently dropped.
  The fix here was to call the merge function with the poisoned directory as its parameter, which has
  no such layer.

## A condition CHANGED is an assertion UN-DEMONSTRATED

An assertion that was demonstrated firing, then had its condition edited, is back to being an untested
rule wearing a tested rule's clothes. The edit is usually made to accommodate a new case, under time
pressure, by someone who already knows the check works — which is exactly the state in which a guard
gets quietly weakened.

**Measured.** A vacuity guard read *"N rows and none of them reached a target cell: the join is broken,
not the file empty."* A new row state was introduced whose rows deliberately reach no cell. The first
edit added that state to the guard's **numerator** — treating one of the new rows as evidence that the
join works. The guard still fired on a totally empty result, still passed on a healthy one, and would
have reported a genuinely broken join as healthy whenever the input happened to contain one of the new
rows. Nothing about it looked like a regression. The correction took the new state out of the
**denominator** instead — *of the rows that SHOULD have reached a cell, at least one must have* — which
preserves the original strength exactly.

**Three arms is the shape for a re-demonstration, and the first is the one that matters:**

| arm | input | required |
|---|---|---|
| A | broken **with the new state present** | RAISE — this is the arm a weakened version passes |
| B | broken **with the new state absent** | RAISE — proof the original condition survived the edit |
| C | the real committed input | **quiet** — a guard that always fires grades nothing |

Arm A is the discriminating one and is the arm most likely to be skipped, because "I already know it
fires on a broken join" is true of arm B and feels like it covers both.

## When a new state satisfies a bucket's TEST but inverts its MEANING, split the bucket

A drift checker bucketed a cell as `RECORDED` when the program named it and a curated artifact had a
row for it — documented as *"expected, and the state that makes a rebuild lossless."* A new row state
meaning *this name is refuted, drop it* satisfies the same literal test (there is a row) and means the
exact opposite. Left folded in, the checker would have booked outstanding cleanup as *"accounted for"*,
in the one instrument whose job is to notice names that are out of step.

The test had stopped being a proxy for the intent. Split the bucket, and let the new count be the
tracker for the work — it should fall to zero when the work is done.

**Report such a count; do not ratchet it.** The neighbouring buckets are ratcheted to 0 because they
are defects. A deliberate-pending-loss count is *expected* to be non-zero until the debt is paid, so a
`max=0` ratchet on it is a gate that fails by design — which is a gate that gets excused, and an
excused gate grades nothing.

## Grade what the artifact CONTAINS, not what a variable holds

An AI-provenance filter check computed the filtered name into a variable, wrote a *different*
expression to the artifact, and asserted against the variable. Bypassing the filter therefore
changed only the output; the assertion compared the same two values it always had. The poison run
printed

```
A4: 1200 of 1947 referring functions are ledgered; the filter changed 1200 of them, 0 leaked
```

— **which is exactly what a healthy run prints** — and failed only on a backstop line at the
bottom. Regraded against the spelling actually written, it raises on 1044 of 1200.

Two things follow. **Run the poison FIRST**, before the clean run makes the output look
familiar; and when it reports nothing, believe it — a poison that produces a healthy-looking pass
is a defect in the check, not a clean program. The general form: **an assertion must read the
artifact's own value, not an intermediate the writer may have diverged from.**

## A census sent for approval is a CLAIM, and its prose is part of it

A repair census printed *"past the end of this struct"* for four records at an offset more than 60
bytes **inside** them. Every cell COUNT in that census was correct and survived unchanged into the
applied run. Only the *explanation* was invented — by an API assumption that had never been
measured — and it was caught because the number was impossible on its face, which is luck rather
than process.

A reviewer uses the prose to decide whether the numbers mean what they appear to. **A wrong reason
attached to a right row is still a false statement someone is being asked to sign**, and it is the
part of a census that nothing else checks: the counts get re-derived by the applier, the sentences
do not. Read them as critically as the figures, and when a sentence explains *why* a row was
skipped or refused, confirm the mechanism rather than inferring it from a return value.

## Record the MARGIN when a gate is cleared narrowly

A repair left four classes modelling 100 of 101 members — 99.0099% — against a threshold of
`>= 99.0`. The gate is green, honestly and by its own rule, and it is green **by 0.01 points**.

State that at the ratchet, with the reason the margin is thin and the change that would widen it.
A threshold cleared narrowly and silently becomes an unstated assumption that the margin will hold;
the next unrelated change in the same range tips it red, and the round that inherits the failure
has nothing to tell it whether it broke something or merely landed on a knife edge that was always
there.


## A fixture written from a description tests the description's author, not the format

When a file format is recovered by reading a binary, the recovery is prose: *"seventeen table
dwords in reverse order"*, *"a count followed by entries"*. A parser is then written from that
prose — and so, usually by the same person in the same sitting, is the synthetic fixture that
exercises it. The two agree by construction. They are one claim stated twice, and a passing suite
measures only that the author was self-consistent.

Measured instance: a container's tamper-protection footer was described as holding
`crc_table[17], [16], … [1]`. The bytes ascend; the phrase described the loader's *backwards scan*.
Parser and fixture both descended, six unit tests passed, and the parser found **zero** footers in
540 shipped files that all have one.

**So the acceptance test for a recovered format is the real artifact, and its expected numbers must
be measured before the parser exists** — otherwise they drift into agreement with it. Practically:

- Take a census of the shipped data with a throwaway script *first*, and hard-code those numbers as
  the corpus test's expectations. That script is not wasted; it is the only independent statement of
  the format you will get.
- Prefer an expectation with **two independent derivations**. A footer count of 181 from a byte
  signature and 181 from counting the records that demand a footer is worth pinning; either alone
  is a single point of failure shared with the parser.
- Where the data cannot be committed (shipped game assets, licensed corpora), gate the test on an
  environment variable naming the install and skip with a printed notice elsewhere. A skipped
  corpus test is honest. A green suite that never touched a real byte is not.

Synthetic fixtures keep their job: they are the only way to demonstrate a *rejection* path firing,
because damaged input is exactly what a real corpus does not contain. Build them to break one rule
each. Just do not let them stand in for evidence that the happy path is right.

## A check that opens with a `continue` cannot police what it skips — write down the skipped population

The blind spot is created by one line and is invisible thereafter, because everything the check
*does* see is verified and looks healthy. Measured on a 1999 MSVC/x86 project:

A gate over the PE export table asserted *"every parsed member symbol carries a declaring class"*,
and opened with

```python
if parse_mangled(row["mangled_name"])["kind"] != "member":
    continue
```

That `continue` skips every **special** name — constructors, destructors, `operator=`, `??_7`
vftables — which on this binary is **111 of 1970 rows**. Behind it, the project's own demangler had
been returning `parse_status="ok"` while leaving `member` empty for **all** of them. The gate ran
green for a whole phase; the committed artifact shipped `CRobotAssemblyBay::` with nothing after the
colons; and two independent investigators eventually hit the blank and demangled by hand rather than
suspect the tool. **The population the check could not see is where the defect had accumulated,
precisely because everything visible was being verified.**

The rule, which costs one comment at the time and is unrecoverable later:

- **When a check begins by excluding a kind, state in the code what the exclusion drops and HOW MANY
  ROWS it is** — as a fraction of the population, never a bare count. "skips non-member kinds"
  conceals it; "skips the 111 of 1970 special names" does not.
- **If the excluded set is a whole KIND rather than a handful of known rows, it needs its own check,
  not an exemption.** The exclusion is usually there because the kind needs *different* handling,
  which is the same reason it needs separate verification.
- **Give the new check a companion that pins its own denominator.** A check of the form "for each row
  where a function exists at this address…" silently becomes vacuous if every row starts taking the
  `fn is None` path. The companion here asserts that the rows *without* a function are exactly the
  `??_7` vftables (data symbols by construction), so the first check's coverage cannot fall to zero
  unnoticed. Demonstrate it alone: re-point one constructor at a data address and only the companion
  may fire.

**And grade the repair against an implementation you did not write.** The expected values here were
never written down in the project — they were read out of Ghidra's own demangler, a separate
implementation of the same Microsoft spec, joined **by address**, agreeing on 96 of 97. The single
divergence (`??B`, a user-defined conversion whose target type is in the *signature*, not in the
special code) was documented *before* the grading run and pinned to that one address, so a second
divergence is a failure rather than a quietly widening allowance. A decoder graded against its own
table has measured nothing.

## A poison that fires NOTHING is worse news than one that fires the wrong guard

The rule "read WHICH check refused, not merely that one refused" has a failure mode below it that
looks identical in a summary line. Measured on one round that added five new guards and nine poison
arms: **three arms hit a guard other than the one they named, and one hit nothing at all.**

| arm | aimed at | actually fired |
|---|---|---|
| "absorbs nothing" | the vacuity rule | the STRADDLE rule — a partly-overlapped neighbour straddles before vacuity is ever reached |
| "claims more bytes than any witness covers" | the extent rule | the NAME-DESTRUCTION rule — the chosen span's uncovered remainder was 1 byte, not 5, so the extent rule did not apply and the arm sailed past it into the names beyond |
| "starts on no witness" | the base-offset rule | **nothing** — the offset chosen as "mid-member" was itself a cell start |

The first two still prove *a* guard works. **The third proves nothing, and it is the one that ships
silently**, because a harness reporting `80 checks, 0 failures` reads the same whether an arm
demonstrated its rule or merely failed to break anything. An `expect_raises` helper catches it only
because it treats "did not raise" as failure; nothing at all catches an arm that raises the *wrong*
exception except a human reading the message.

**So: read the raise text of every arm, every time, and treat an arm whose poison turned out to be
valid input as an UNTESTED rule rather than a satisfied one.**

Two constructional sub-rules, both paid for in that round:

- **A poison for a "this region is too big / has unexplained bytes" rule cannot be built out of
  PADDING.** Padding is bounded by the alignment maximum *by definition*, so a span made of it can
  never exceed the threshold. It needs an explicitly-*unknown* region.
- **A poison for "starts on nothing" needs a GAP, not an interior offset.** In a gap-free tiling
  every interior offset is some cell's start. The intuition "the second of four components is
  mid-member" is false precisely when the four components are four separate cells — which is the
  case the rule exists for.

## Making an artifact more EXPRESSIVE enlarges what an existing check can grade

A check that grades "every row with a decided name" has a denominator that moves whenever naming
moves. Measured: a round taught a layout artifact to express grouped members, which named three
previously-anonymous offsets; a descendant-drift probe that had been green immediately reported
three faults against a ratchet ceiling of zero.

**Nothing had regressed.** Those three offsets had never been in that probe's population, so it had
never been able to look at them; the faults were pre-existing, and two other gates were already
reporting the same three cells.

Both of the obvious readings are wrong and expensive. Reading it as a regression raises the ceiling
and buries three real faults. Reading it as "my change broke something" reverts a correct change.

**The discriminator: does the newly-graded population OVERLAP what another gate already reports?**
If yes, it is newly visible, not newly wrong. If no, it is a genuine finding and deserves its own
investigation.

Practical consequence: **state a check's denominator in the round's write-up whenever it moves, not
only when its verdict does.** In that incident the probe had been printing `grading the 82 with a
DECIDED NAME` all along, and the number had been 74 the previous day. The information was on screen
and nobody was watching that column.

## A poison arm can be undemonstrable against the POPULATION, not the harness

*"A poison that reports `did not fire` is a claim about the HARNESS first"* is right, and it has a
second reading that costs a round if you miss it: sometimes the harness is fine and **the arm is
unreachable on this particular data**. Three arms failed in one round, each for a different reason,
and only the third looks like an obvious mistake:

1. **The branch is unreachable on this population.** An arm loosened a rule to accept a wrong witness
   shape (a register used as a *scaled index* rather than a base pointer) and moved nothing — because
   every function that ever reached such an operand had already satisfied the rule legitimately
   earlier in the same body, so the loosened branch was never evaluated. **That is not a safety
   argument for the rule.** It is a fact about the sample, and it means the arm proves nothing here.
   Delete it, and record why, or a later round re-proposes it.

2. **The pinned count SATURATES.** The arm targeted *"does at least one witness qualify?"* — a
   boolean that is already true. Accepting extra bogus witnesses cannot move a value at its ceiling.
   **Poison a count that can move**: the total over every classification, not the per-subject
   any/none. Pin both; the boolean is what the verdict uses, the total is what the poison can reach.

3. **An apply emptied the arm's target.** The arm flipped a `repair` verdict; after the apply there
   were no `repair` verdicts left, so it silently became a no-op. **Re-demonstrate every arm after an
   apply changes the population** — an arm demonstrated pre-apply is not demonstrated post-apply, and
   the meta-check ("poison run passed") is the only thing that catches it.

The surviving arm poisoned the rule that actually decides rows, and moved three separate pinned
counts. That is the shape to aim for: **an arm should break the thing the verdict depends on, not a
thing adjacent to it.**

## Read the WITNESS a rule printed, not the count it produced

A rule that identifies evidence must print the specific item that satisfied it, and you must read a
sample. Measured: a rule scoring a healthy-looking **26 of 27** was citing, as proof that a register
held an object pointer, instructions that *overwrite* that register (`LEA ECX,[ESP+0xc]`, `POP ECX`,
`IMUL ECX,EAX`) and one where it was a scaled table index. The count looked right; six of the reasons
behind it were wrong. Nothing but reading the printed instructions could have shown that — no
assertion was violated, no total was implausible.

The correction has its own trap. The fixed rule landed on **the same rows** as the broken one, so the
numbers had been right by luck the whole time. **A rule reaching the right answer is not evidence
that it reaches it for the right reason**, and only the per-item witness distinguishes them.

Over-correction is the mirror image and just as easy: the first fix here rejected the target
platform's *standard idiom* for the thing being detected, and quietly discarded a third of the real
evidence. When you tighten a rule, check what it now REFUSES, with the same sampling.
## Keep a replaced rule in the file, wired to its own poison arm

When a rule that produced *committed results* is found unsound and replaced, deleting it is tidier
and worse. Keep it, unused, as a poison arm.

Measured: a similarity-based grouping rule was replaced by one based on recovered parentage. The old
rule stayed in the file behind a `poisonglobal` flag. The arm is unusually good precisely because it
is not a reconstruction — it is the original — and injecting it moves four separate pinned counts
*and* reproduces on demand the specific false positive that motivated the replacement. A reader
comparing the two functions sees exactly what changed and why.

The general shape: **a poison arm modelled on a real historical defect cannot drift from the thing it
models.** A hand-written "wrong version" can, and usually does, become subtly easier to catch than
the mistake it stands in for.

## An adjudicated exception is a NUMBER, not a category

A round found one already-applied row whose evidence no longer held, and the decision was to record
the downgrade rather than revert it. The naive way to encode that is a check that tolerates rows in
that state — which then waves through every future one, silently, forever.

Instead pin the **count**, with the address in the constant's comment:

```python
EXPECT_APPLIED_UNWITNESSED = 1     # 0xNNNNNNNN Class::Method, slot 109 (§284)
EXPECT_APPLIED_UNTESTABLE  = 1     # 0xNNNNNNNN Class::Method, vftable unmapped
```

The check does not raise on the adjudicated row; it raises the moment either population **changes
size**. A second such row then stops the gate instead of joining a standing exemption. **Never widen
the constant to make a gate pass — adjudicate the new row.**

## Split a status that has come to mean several things

One status had accreted three outcomes needing opposite responses: *still supported*, *supported by
evidence this instrument cannot examine*, and *support disproved*. Collapsed into one label, the only
one that mattered was invisible; the fix was three verdicts, not a footnote in prose beside the
artifact. **When adjudication finds a status covering outcomes that call for different actions, split
the status.**

## Make the control group the nearest neighbours, not the leftovers

A first round used its own abstentions as the control for its apply. The follow-up used *every row
not being applied* — which now included the rows the first round had already treated. Those are the
closest possible neighbours of the treatment group in class, shape and provenance, so they are where
a stray effect of the apply or its cascade would actually show. **Prefer a control group that has
already received the same treatment over one that was merely never eligible.**

## Where two artifacts partition a population, assert the partition

A sweep drew its population from two artifacts — "still outstanding" and "already done" — which are
disjoint by construction. Run in the wrong order after an apply, three addresses appeared in both. A
one-line intersection assertion named them; without it the population would have carried silent
duplicates and surfaced rounds later as an unexplained count. **Assert the partition, not just the
total.**


## A gate's premise about ITSELF goes stale like any other claim — and it lives where no audit looks

A stability harness that re-runs your producers and byte-compares their artifacts is usually a
project's most trusted instrument. It is also the one whose *own* assumptions nobody re-derives,
because everything it reports is about something else.

**Measured.** A harness drove two populations: `SWEEPS` (which write artifacts) and `PROBES`
(asserted on their printed verdict). It ran the probes **after** taking its comparison snapshot,
computing the artifact diff and computing coverage — on a premise it printed on every pass:
*"driven probes … they write no artifact, so the diff above cannot see them."* True when written.
Then one probe began writing an artifact, and the sentence became false inside the instrument that
audits everything else.

- **REGISTERING SOMETHING NEW WITH A GATE IS A TEST OF THE GATE, AND IT IS FREE.** The intent was
  bookkeeping. The harness answered `REACH FAULTS: 1 committed CSV … neither regenerated nor
  excluded` in the same run that printed `wrote …(120 rows)` twenty lines above. Nothing about the
  artifact was wrong. The only way to find the defect was to hand the gate a shape it had never
  been handed.
- **THE LOUD HALF AND THE SILENT HALF HAD OPPOSITE SIZES.** Loud: one coverage fault, refusing the
  whole run. Silent: because the write landed after the snapshot, **the artifact was never
  byte-compared at all** — a probe writing something *different* would still have produced
  `0 CHANGED`. The half that shouted was the harmless one. Ask of any gate reporting coverage:
  *what does it do with a thing that arrives after it has finished measuring?*
- **THE WRONG FIX WAS ALSO THE UNDETECTABLE ONE.** Adding the artifact to the "deliberately
  excluded" table would have silenced the fault in one line, and that table's self-retiring guard —
  *an exclusion whose artifact IS regenerated raises* — could not have caught it, because under the
  old ordering the artifact was never counted as regenerated. **A self-retiring exclusion does not
  retire when the thing that would retire it runs too late to be seen.**
- **FIX BY ORDERING, NOT BY A SECOND DIFF.** Running the probes *before* the snapshot puts both
  populations under one diff and one coverage arithmetic. Re-snapshotting afterwards and folding the
  result in would have been a second copy of the diff logic — the "one home for a rule" failure this
  file exists to prevent. When moving a block inside a large gate, anchor on CONTENT rather than
  memorised line numbers, and assert that the moved block references nothing defined in the section
  it jumped over.
- **AND THE CLAIM WAS INVISIBLE TO THE PROJECT'S OWN CLAIM AUDITOR**, which read `#` comment lines
  in `.py` and never string literals. A claim about a tool, emitted *by* that tool, inside a
  `print(...)`. If you have an instrument that hunts unbacked claims about your tooling, ask what
  syntax it cannot see before trusting its zero.

**A ground-truth refusal beats a constructed poison here.** A poisoned copy restoring the old
ordering was built and discarded unrun: the first live run had already demonstrated the failure, on
real data, for the real reason.

## Share an implementation; never share the number a check is graded against

A gate and the probe it trusts frequently need the same calibration constant — a floor, a ceiling,
an expected population. Importing it from one to the other is the instinct, and it is wrong.

**Measured.** A premise gate rested on a probe's calibration (*21 of 25 ground-truth rows must
classify a certain way*). The gate defines `CALIBRATION_FLOOR = 21` **as its own literal**, a second
definition of the probe's. Importing it would make the two agree *by construction*: the gate would
follow the probe silently wherever the probe moved, and a probe re-calibrated downward would drag
its own gate down with it, still green.

Two independent literals that must agree convert a drift into a **disagreement somebody has to
adjudicate**. This is the deliberate exception to "one copy of a rule on disk": share the code that
computes, never the threshold the computation is judged against.

## A refusal that FIRES is not a refusal that fired for its REASON

A guard that refuses when it should is usually taken as proven. It is not: it has been shown to
produce the right *verdict*, which is a much weaker claim than producing it for the right reason.

**Measured.** A batch applier compared an approved change-list against the rows that would actually
fire and raised on any difference. It refused a deliberately-shortened list correctly on every run —
while its parser was returning `('', '', '<- witness…')` for every row. It was comparing garbage
against garbage and reaching the correct answer. A test asserting *"this run refused"* would have
passed forever, and so would the exit status.

- **Assert on the refusal's CONTENT** — the addresses, the values, the reason string — not on the
  fact of refusal. That is the only thing separating a working guard from a coincidence. A harness
  that prints what a failing probe *said* is the same instinct one level down; this is the version
  that belongs in the arm itself.
- **A COUNT-BASED VACUITY GUARD IS BLIND TO A SHAPE CHANGE.** The parser above asserted it had
  recovered as many rows as the header claimed. When the format grew from one line per row to two,
  it began counting one *witness* line per row — the count matched exactly while every field was
  wrong. Guard the SHAPE (this field is an 8-hex address, this one is a registered enum value, this
  one is non-empty), not the tally. A guard on how many is blind to what.
- **Both defects here scored green on exit status**, and both were found by reading text. Where a
  check's output is the evidence, a run nobody read is a run that did not happen.

## Changing a RENDERING is an interface change, and the consumer breaks silently

Text output that another component parses is an interface, whether or not anyone wrote one down —
and it is the interface people forget is one, because it has no signature to break.

**Measured.** Removing a truncation from a census renderer — a correct, necessary fix — silently
broke a parser in another component within the hour. The sweep that would have named the victim in
advance was one command: grep for the function symbol, which returned exactly three files.

**Before changing any rendering another component reads, enumerate its consumers and say what each
one does with it.** Then change them together. The same rule as "a rule change in a producing sweep
needs an old-vs-new diff", pointed at the one output nobody thinks of as data.

## A status flag that conflates two failure kinds makes its consumers diverge

When a result object carries a boolean `passed` and a list of `refused` items, the boolean will be
false for two structurally different reasons, and independent consumers will read it differently.

**Measured.** A category gate refused both when its *premise* collapsed (false for every row that
rests on it) and when a single row was *inadmissible* (false for nothing). One consumer read
`passed=False` as "drop the whole category"; another dropped only the listed rows. Neither reading
is unreasonable, and the divergence was expensive in two directions: one bad proposal would have
discarded 432 good ones, and because the two consumers must produce identical sets for a drift check
to pass, a perfectly healthy batch could never have been applied at all.

**The fix needed no new field.** The item list was already correct for both cases, because a premise
failure is expanded to every row *by the gate that knows it failed*. So:

- **the list is the decision; the boolean is for reporting** — say so in the docstring, because the
  next consumer's author will otherwise re-derive the wrong reading;
- **then make it a contract**: raise if a result is `passed=False` with an EMPTY list. Otherwise a
  future category's premise failure reads as "nothing to drop" and the whole category is APPLIED —
  the failure in the direction that writes to the program.

**And note the shape, because it recurs**: two components deriving the same population by different
routes, with nothing asserting they agree. Where two components must compute the same set, one
should CALL the other, or a check should assert the two agree on real data — and that check must
then be tested for firing on the right *reason*, per the section above.

## A gate ratifies a guess when the guesser and the gate read the same way

The standard defence against a bad automated proposal is a gate that re-derives the claim from the
binary and compares. It is a real defence, and it has one hole that no amount of gate-hardening
closes: **where the underlying evidence is ambiguous, the producer's guess and the gate's
re-derivation will usually make the SAME guess**, because they scan the same bytes in the same
order. The gate then ratifies it, and a guess ratified by a checker sharing its bias is
indistinguishable from a measurement.

**Measured.** A generator emitted a name for each compiler-generated initializer, derived from the
global that body writes. 106 of 441 bodies write more than one global — one writes 144. "Take the
first store" would have produced a plausible name for every one of them, and the gate, taking the
first store too, would have agreed. The generator instead **emitted nothing and recorded the address
and the reason**; 122 of 489 candidate rows died there. That refusal count is the wave's most
valuable output, not its shortfall.

- **Where a body admits more than one derivation, refuse.** A smaller correct wave beats a larger
  one carrying invisible guesses, and the deferrals are a queued backlog with addresses rather than
  a worry.
- **Make the producer INDEPENDENT of the gate.** If the producer imports the gate's helpers,
  "N of N passed" is a tautology — the gate is confirming the producer's own arithmetic. Share the
  schema and the file format; never the rule that turns evidence into a claim.
- **Give the gate a question whose answer you already know.** Do not generate over the pre-filtered
  population expected to pass. Generate over the WHOLE one and predict the refusals: here the
  calibration named nine addresses that must be refused, the gate refused exactly those nine, and
  the producer had predicted the same nine independently. That turns a green run from *"nothing
  objected"* into *"two separately-written readers agree on the accepts AND the rejects"*.
- **`input = emitted + skipped`, asserted, is the only thing that sees a FALSE NEGATIVE.** A gate
  judges only what was emitted, so a too-narrow pattern is invisible to it: the wave quietly does
  less than it should and reports success. Make the arithmetic raise, and check that each leg is a
  measurement — one leg here defined its population by the same predicate that later emitted, so
  `36 = 36 + 0` was an identity that could never fail.

## A dry run that skips a stage rehearses only the parts that were already easy

A mutating tool's dry run exists so the apply holds no surprises. It earns that only if it executes
every stage except the write.

**Measured.** A batch applier's dry run stopped before the signature parser. The parse was therefore
the one failure class discoverable only by attempting a real apply — and that is exactly how it was
discovered, with the whole batch rolling back. The fix is one line of ordering: parse and translate
every row during the dry run, discard the result, and report `N of N parsed`.

**And the root cause was upstream of the parse, in a place no failure pointed at.** A constant
`PROTO_ARGS = (True, False)` sat under a comment naming the API's second argument
`includeCallingConventionOverride` — **a parameter name that does not exist**. The real signature
took `(formalSignature, includeCallingConvention)`, and `formalSignature=True` silently omits
auto-parameters. Two consequences, neither visible from the failure: one guard was structurally
dead, and the pre-mutation snapshot — the round's only way back — had been recording an unfaithful
"before" state. **A comment asserting an API's shape is a tool claim like any other**, and it is one
a claim-auditing sweep will miss, because it names nothing checkable.

The compensating design worth copying: the failed apply wrote **nothing**. One transaction, 367
writes, atomic rollback. Concentrating mutation in a single bracketed applier — rather than
spreading it across per-round scripts — is what made a defect in the mutating component cost one
re-run instead of a program to repair.

## A schema field nothing READS is a check you believe you have

A gate can be missing in a way no gate detects: the *data it would grade* is collected faithfully,
round after round, and no code ever opens the column.

**Measured.** A naming catalog carried a `prediction` field — described in every delegated reader's
brief as *"a falsifiable consequence of your name being right"* — for **987 rows across 13 rounds**.
The identifier appears in the catalog's verifier exactly once, **inside the schema header list**. No
checker touches it. Eight other scripts in the same pipeline carry it through a header list or a
plan tuple and write it back out unchanged. The project meanwhile made predictions gate-bearing in
three *mutating* appliers, one of whose headers reads *"the prediction is the TEST — reconcile it,
do not adjust it to match"*. The pipeline that collected predictions was the one place that graded
none.

**The reason it survived thirteen rounds with every gate green is the part worth keeping:**

> There is no observable state in which an **ungraded** field differs from a field that is
> **checked and always passes**. Schema checks, encoding checks, ledger joins and byte-identity
> canaries all pass on a full column exactly as they would on a graded one.

So the defect is invisible to every instrument *except* the question *who consumes this column?* —
which nothing asks on a schedule. Two cheap habits close it:

- **When you add a field to a schema, name its consumer in the same commit**, or write
  `# NOT GRADED` beside it. An ungraded field is legitimate; an ungraded field that reads as
  graded is not.
- **Periodically invert the artifact/consumer map.** A tool that answers *"which script writes
  this CSV"* is usually already present; the useful direction is the other one, per column.

**And check what the collected values actually claim before switching a check on.** Of the 912
`decided` rows above, **586 (64%)** carried one boilerplate sentence — *"this body still references
**the cited** string and **the cited** global"* — which restates the row's own citations, i.e.
exactly what the citation checker already re-verifies every pass. Grading them would have graded the
population against itself. **A prediction that names only what the row already cites is a
restatement wearing a prediction's column**, and it is the guesser-and-gate-read-the-same-way
failure moved one column over.

**The corollary that pays.** The remaining 243 rows had converged, unprompted, across **seven
separate rounds**, on one shape: **exclusivity** — *"no other body calls this import"*, *"the ONLY
non-vtable caller"*, *"never reached by any other class"*. That is the right shape for a structural
reason, and a grammar invented at a desk had missed it: a citation says *this body references S*,
and the falsifiable consequence of the **name** is *and nothing else does* — **a whole-program query
the reader could not run and its evidence pack did not contain.** When you finally build the checker
for a long-collected field, **derive its grammar from what the producers already wrote**, not from
what a clean design would prefer. They have been solving the problem for longer than you have.

## A 100% pass rate is a corroboration only after you measure the BASE RATE of violation

A check that returns *N of N pass* is reporting one of two very different things, and the output
looks identical either way: *"the population genuinely has this property"*, or *"this property is
structurally impossible to violate here"*. The second is an unfireable arm with a healthy-looking
number in front of it.

**Measured.** A predicted claim — *"slot 38 of this class's vtable is never reached by any other
class"* — was checked over 203 catalog rows: **203 OK, 0 FAIL, 0 unresolvable.** Clean sweeps are
exactly when to stop and ask. The follow-up query was one line over the same artifact: of the
**1,779** distinct vtable targets in the program, **708 (39.8%)** appear in more than one vtable. So
the claim is violable by two functions in five, the 203 sit in the satisfying minority, and the pass
is a real corroboration. Had the answer been *0 of 1,779 are multi-table*, the 203 would have been a
tautology, and the check would have been reporting health it never measured.

- **The base rate is usually one aggregate over the artifact the check already reads.** It costs
  nothing next to writing the check, and it converts *"nothing objected"* into *"the property held
  where 40% of comparable cases would have broken it"*.
- **Print it beside the verdict**, the same way a vacuity guard prints `empty=` and `unexercised=`
  as values. A reader who sees `203/203` cannot tell; a reader who sees `203/203, base rate of
  violation 39.8% over 1779` can.
- **The demonstration still matters, and prefer ground truth to construction.** Here the same claim
  asserted for four *real* multi-table bodies — one of them reached from **225** vtables — returned
  `PASS=False` on all four, which is a stronger arm than the constructed poison beside it because
  nothing about it had to be fabricated.

**A measured zero from the same sweep is worth keeping even when nothing needs it.** The base-rate
query above also returned *0 of 1,779 targets appear at more than one distinct slot **index***, i.e.
a function occupies one slot number across every table that holds it. No claim depended on it; it is
a whole-population invariant that cost nothing to observe and would cost a round to re-derive.

## A comment describing a branch is not a branch

The same round found the inverse failure — a query tool that **answered where it should have
refused**, with exit 0.

**Measured.** A corpus exporter wrote two of its program-wide indices as empty lists, under a
comment reading *"a bounded trial writes them EMPTY rather than partial, because a partial index
reads exactly like a measured absence."* The reasoning is correct and **the branch it describes does
not exist**: the full export writes them empty too. The two CLI commands built on those indices then
printed nothing (exit 0) and `no recorded writer for <addr>` (exit 0), for a string with three real
referrers and a global with real writers — three of the seven evidence kinds the whole pipeline runs
on, silently unserviceable.

> A comment asserting a code path is a tool claim like any other, and it is one a claim-audit sweep
> will miss, because it names nothing checkable. **Grade the branch, not the comment**: if a
> conditional matters, there is a run of the tool in each arm that proves it.

And for anything that answers queries over an index: **an empty index must produce a refusal, not an
answer.** The rule is the same one that governs a stale stamp — print what to do instead of
returning a result the caller cannot distinguish from a measurement.

### Two sharpenings of the base-rate rule, from the round that first used it

**1. A base rate measured over the population you selected FOR passing is the vacuity it was meant
to detect.** Measured: the first cut computed one form's violation rate over *the addresses the
artifact cites* and reported **0.0%** — which is not a base rate, it is the checked rows passing,
restated with a percent sign. Over the comparable population (every string any body references) the
answer was **9.2%**. The denominator must be the population the checked rows were drawn FROM, never
the checked rows.

**2. A claim can be TRUE and still not worth checking, and retiring it is a result.** One form was
seeded, evaluated and withdrawn inside a single round. Its rows failed; adjudicating them showed
every failure was the *checker's* fault, not the reader's. The obvious repair was then **measured
before being written** and turned out to be satisfied by 307 of 312 comparable cases — a 1.6%
violation rate — because the behaviour it tested for *is the language idiom the claim describes*, so
the claim's own exception clause swallowed its assertion. It was retired rather than shipped weak,
with the rule kept in the file wired to its own arm.

> Ship the measurement, not the arm. A check at a 1.6% base rate spends real runtime to report a
> property nothing was going to violate, and it does it wearing the same green as a check that means
> something.

## Writing assertion arms is not knowing they can fail — mutate the source

Arms are written against the failures the author imagined. The ones that matter are the guards that
looked too simple to need an arm.

**Measured.** Ten deliberate one-line breaks to a freshly written, heavily-armed module: **nine
fired the arm written for them, and one — deleting a duplicate-row guard in the loader — was not
detected at all**, with the selftest still reporting *34 of 34 pass*. Three further breaks made an
arm raise a domain exception rather than fail its assertion, so the harness printed a **traceback**
where it should have named the broken check; the selftest still exited non-zero, so the mutation was
caught, but the report answered *"something broke"* where the whole purpose is *"THIS check broke"*.

Both are cheap to close and neither is visible without mutating:

- **The arm-runner catches every exception, not just `AssertionError`.** Anything narrower converts
  an informative failure into a stack trace at exactly the moment you need the name.
- **Mutate one guard at a time and record which arm fires.** An unfired mutation is a missing arm;
  a mutation that fires three arms is usually correct (one guard, several consequences), and a
  mutation that fires an *unrelated* arm is a coupling worth knowing about.
- Do it **when the code is written**, not later. The undetected guard above was in the function that
  looked least risky, which is why it had no arm and why nothing else would ever have found it.

### The mutation harness is source too, and it is wrong first

The most convincing mutation report is the one where the harness itself is broken: **every mutant
"fails", nothing is measured, and the run prints `7/7 detected`.** Measured — seven mutants of a
producer were written to a temp directory and every one exited non-zero on `FileNotFoundError`,
because the script derives its repo root from `__file__` and the copy was outside it. Not one arm
had fired.

Two lines close it, and they turn the harness into an instrument that can report its own failure:

- **Run the mutants where the original runs**, and **assert an UNMUTATED copy at that same path
  PASSES first.** A baseline that fails means the harness is broken, and every result after it is
  noise.
- **Require the failure to come from a NAMED ARM, not from a non-zero exit.** Grep the output for
  the arm's own label. A crash, a usage error and a fired assertion all exit non-zero, and only one
  of them is evidence.

Do the same for the arms themselves once they pass: a mutation that fires an arm you did not aim at
is telling you the arm you *did* aim at is not testing what its name says.

### A presence check over a two-source join tests the UNION, not the source you meant

A row assembled from two witnesses — two decoders, two sweeps, a probe plus an artifact — appears in
the output if **either** contributed it. So `X is in the result` is an assertion about the union, and
it stays green while the source you actually care about goes silent.

**Measured.** A producer joined its own parse against a second decoder's and tagged each row
`both` / `a_only` / `b_only`. Two arms asserted that six specific classes "survive", by membership.
Mutating the parse to drop all six left both arms green: the rows were still there, now tagged
`b_only`, and nothing looked at the tag. The arms now assert `witness == both`, and the mutation
that motivated the tightening is what fires them.

The general form: **assert the row's PROVENANCE, not its existence**, wherever a row can arrive by
more than one route. The tag already exists — a join that classifies its rows and then checks them
without reading the classification has built the instrument and thrown away the reading.

### A vacuity guard written for the poison case will fire on the live one

Guards that exist so a poison arm has something to trip are usually described as ceremony. Two
measured cases say otherwise, and in both the guard fired on **real input, before any poison ran**:

- A cross-check probe called `DemanglerUtil.demangle(program, name, addr)`, which on Ghidra 12.1.2
  returns a `java.util.List` rather than a `DemangledObject` — so `getDataType()` was simply absent
  and the probe reported a decoded type for **0 of 979** symbols. That reads as *"the second decoder
  has nothing to say about these"*, which is a finding, rather than as *"this is the wrong entry
  point"*, which is a bug. `if decoded == 0: raise` said so in one line.
- A sweep whose whole product is an agreement count, run against an artifact that had not been
  regenerated: it compared nothing and would have reported health.

So write the guard for the shape of the answer — *"this producer's entire output is a comparison; a
comparison of zero things is a broken instrument, not a clean result"* — and let the poison arm be
the demonstration rather than the reason.

### A schema or a check that embeds a fact the pipeline routinely rewrites will go stale silently

Ask of every stored string: **which part of this does a later round routinely change?** That part is
a fact about the program, not about the check, and freezing it inside the check turns the check into
a claim that the program never moves.

Four measured instances in a single round, all the same species:

- a gate comparing a **rendered prototype** as a whole string — the rendering embeds the function's
  own name and the `this` parameter's type, both of which a naming round rewrites, so twelve
  correctly-unchanged signatures were reported as partial reverts;
- an evidence pack citing a neighbour **by name** where the name may still be the disassembler's
  placeholder;
- a template set embedding an **input format** the producer had since changed;
- an example in a brief writing an integer in **hex** where the grammar's type is decimal — and five
  independent readers copied the example rather than the type.

The last one generalises past schemas: **an example in a brief IS a specification.** Readers follow
the example.

### A check nothing runs is a premise decaying unobserved

A probe that is in no harness — no sweep list, no gate, no CI — still contains assertions, and those
assertions are claims about a program that keeps changing. It does not fail quietly; it does not
fail at all, because nothing invokes it. The next person to run it finds a raise, and cannot tell
whether they broke it.

Measured: seven producers were edited in one round and run by hand afterwards. One raised —
*"`CBasicGobject +0x1c4` is now named `TargetHandle`; the gap this probe measures has been closed by
some other route"* — an assertion working exactly as designed, about a fact that had changed some
rounds earlier. It was in neither of the two lists the harness drives.

- **Registering a probe is what keeps its premise honest.** If it is too expensive to run every
  pass, excuse it *by name with a reason*, so the exemption is visible; an unlisted probe is neither
  covered nor excused, it is invisible.
- **When you touch a script the harness does not drive, run it, and run the PRE-EDIT copy too.**
  `git show HEAD:path` into a scratch name, run both, diff the output. Otherwise a pre-existing
  failure becomes yours and a failure you introduced becomes "it was already like that" — and both
  mistakes are made in the same direction, toward the answer that costs less.

### A licence recorded in a round's WRITE-UP is not a licence the next round inherits

The dangerous form is not a missing check. It is a check that was **run once, by hand, and
reported in prose** — because the prose reads like the property holds, and the next round
inherits the conclusion without the test.

Measured. An applier relabels a calling convention on the argument that two conventions pass
their first parameter in the same register. True of the *convention table*; a claim about a
*body* only if that body has no argument in the register the two conventions do **not** share.
One round knew this, ran a liveness probe by hand over its 13 targets, and wrote *"13 of 13
dead"* in its write-up as the independent witness the original argument lacked. Nine rounds
later the same producer issued a fresh census, adjudicated **without** it. Re-run: of six
proposed repairs, **four clean, one refused, one actively contradicted** — a third of the census
resting on a verdict nobody had asked for.

- **If a hand-run check licensed an apply, the next apply must not be able to skip it.** Emit the
  verdict as an artifact and have the producing rule CONSUME it. A precondition that lives in a
  round's notes is a precondition with a one-round half-life.
- **Have the producer and the checker read the bytes differently.** If the rule that proposes and
  the gate that grades both compute liveness the same way, their agreement proves nothing.
- And when you propose the apply: **read what the PREVIOUS round used to license the same
  operation before writing the census**, not after. In this incident a census of six went out for
  approval on body reads alone and two rows had to be withdrawn.

### `if not X` catches an EMPTY join, never a MISMATCHED one — count what matched

A lookup between two artifacts fails in two ways, and the guard everyone writes catches only the
harmless one. `if not TABLE: raise` fires when the file is empty. It cannot fire when both sides
are fully populated and simply **do not meet** — different address spelling, different case,
different zero-padding — and that is the failure that reads as a clean result.

Measured, caught by testing the join before wiring it up rather than after: one artifact spelled
addresses `0x00411460` (a shared normaliser), the consuming sweep keyed on the disassembler's
own rendering `00411460`. **134 keys on each side, intersection 0.** Every row would have
received no verdict, every proposal would have demoted for want of one, and the sweep would have
printed a healthy `0 repair, 146 abstain` — a census emptied by a string format, with no error
anywhere in the pipeline.

- **Assert the MATCH COUNT, not the presence of the table.** `matched == expected`, and say what
  a partial match means, because a partial match is two censuses drifting apart.
- **Normalise on BOTH sides through one function**, and reach for the project's existing
  normaliser. In this case its own docstring already named the failure — *"a lookup that silently
  misses and reads as no such function"* — in a module the call site had chosen not to import.

### The brackets that matter at the moment of a mutation are the ones nothing checks

Pre-flight tooling tends to grow around the **read-only** harness, because that is what runs every
pass. So the invariant constants inside *appliers* — the ones asserting the program's state
immediately before it is changed — are exactly the ones no routine check covers.

Measured: a pre-flight checker reported *"2 bracket constant(s) checked, 0 stale"* while **all five**
of a mutating applier's pre-state constants were stale by an intervening round's deltas. The
applier would have **refused** rather than mis-applied, so the design held; the defect is that the
staleness was invisible until a round happened to open the file.

- **Enumerate bracket constants across mutating scripts too**, or state in the checker's own output
  which population it covers, so "0 stale" cannot be read as "0 stale anywhere".
- **Re-pin with a named account, never by subtraction.** Each delta should cite the round that
  caused it and a committed artifact that corroborates it — `+188 symbols` is one round's applied
  names, checked both directions by the ledger gate; `+2 datatypes` are two placeholder structs
  named in that round's own record. A re-pin without that account is a re-baseline.


## A guard that succeeds destroys its own positive control

A probe that grades a guard cannot be calibrated on the guard's output. This sounds obvious
stated that way and is invisible in practice, because the calibration is usually written
*before* the guard works, when the thing it looks for is still there.

Measured. An exposure probe required a specific applied type name to be findable in a
committed evidence column — a drift a previous round had genuinely measured, so the
calibration was real when written. The producer of that column then gained a suppression
filter for exactly that class of name. From that moment the calibration could only fail: the
thing it looked for is the thing the producer now removes. It went unnoticed because the probe
was also raising for an unrelated reason, so nobody ever reached the calibration.

The obvious repair — point it at a *different* committed artifact whose whole job is naming
such types — also failed, and the failure is the general case: measured across **every**
committed column, once the guard worked, **no** column of that kind carried a qualifying name
except the handful the probe was there to report. A successful guard drives its own positive
control to zero everywhere.

So the control has to be **constructed**, and constructed controls have a failure mode of
their own:

- **Synthesise from real data through the real code path.** Take a genuine name from the
  ledger, render it in the actual format of each column being scanned, and run it through the
  same tokeniser and matcher the probe uses. A control that exercises a simplified path proves
  the simplified path works.
- **Make it TWO-SIDED.** A matcher that returns everything passes a positive-only check exactly
  as well as a correct one. Assert both that a real name IS found and that a name in no ledger
  is NOT — per scanned column, so the count is a fraction of the surface (`5/5 found, 5/5
  rejected`) rather than a bare "calibration ok".

And the diagnostic worth keeping: **when a calibration starts failing, ask whether the world
changed or the instrument did.** Here the world changed, correctly, in the direction the
project wanted — and the check that was supposed to notice regressions had been quietly
measuring a quantity that success was designed to eliminate.

## Three regex answers to one question means the question needs a parser

A completeness check that asks *"which scripts write a file?"* by pattern-matching source text
will be wrong, and widening the pattern is not the fix.

Measured on one project, on a single producer that had to be found:

- the check scanned with `os.listdir(dir)` rather than a walk, so an entire subdirectory —
  164 files, a third of the tree — did not exist to it;
- its write-pattern matched only the project's own two CSV helpers, missing a hand-rolled
  `csv.writer`;
- and the producer wrote as `import csv as _c` / `_c.writer(`, so **widening the pattern to
  `csv\.writer\(` still missed it.**

Three shots at the same question by pattern, three wrong answers — and the project already had
an AST-based resolver for exactly this question, written earlier *because three regex copies of
it had disagreed with each other*. Reuse it. An `ast` walk answers "does this module CALL this
method" without confusing a mention in a comment for a call, which the regex also got wrong in
the other direction.

**The corollary that bites during the fix:** an AST resolver honestly returns a *glob* when an
output filename is computed at runtime. Matching its answers against real filenames with a set
intersection then silently drops every such producer. Match with `fnmatch` when the name
carries a metacharacter — and note what surfaced this: the stale-entry arm had just been
changed from a `print` to a `raise`. **A note printed by a check nobody runs is not a signal.**

## A witness that agrees with BOTH hypotheses is not a witness for either

Auditing your witnesses for INDEPENDENCE is not the same as auditing them for DISCRIMINATING
POWER, and a round can do the first impeccably while skipping the second entirely.

Measured on a 1999 MSVC/x86 binary. A probe was written to decide whether 224 bodies at a
fixed vtable slot were the class's **vector** deleting destructor (`??_E`). Everything about
its design is right, and it is worth reading as a model:

- it opens by REFUSING to extend a 9-target claim to 203 on artifact-side evidence alone,
  on the stated grounds that the artifact-side observations "all reduce to *the vtable is
  shaped the way a destructor slot would be shaped*" and would be the project corroborating
  its own layout work;
- it names three BODY markers as the independent witness — the `__flags & 1` deallocation
  test, `RET 4`, and a call to the deallocator;
- and it **discovers the deallocator by counting callees rather than hardcoding an address**,
  explicitly so that "the leg that could most easily have been an assumption is instead a
  discovery confirmed by ground truth". One address was called by 224 of 224 bodies and only
  then looked up — it carried an export symbol, i.e. the binary's own statement.

It reported 224 of 224 on all three legs, and the round named 203 functions
`<Class>::vector_deleting_dtor`. **All 203 were wrong.** The three markers are carried by the
vector deleting destructor *and* by the scalar one (`??_G`) alike. The probe could only ever
confirm *"this is a deleting destructor"*; it was read as confirming *"this is the VECTOR
one"*. The discriminator is one instruction it never looked for: MSVC's deleting-destructor
flag byte has bit 1 (`& 2`) selecting the ARRAY path — read the element count the compiler
stored at `[this-4]`, destroy each element in a loop, free from `this-4`. The probe's matcher
accepted an immediate `1`/`0x1`; the string `2` appeared nowhere in it. Re-measured across all
258 named bodies afterwards: the array bit occurs **9 times**, in exactly the 9 the earlier
round had read by hand and named correctly. The split fell exactly on a round boundary, with
no round mixed — the signature of a LABEL being copied forward rather than a shape re-read.

**The check, and it costs one question.** Before adding a fourth corroborating leg, write down
the rival hypothesis and ask *which of my legs is FALSE under it*. If the answer is "none",
the round has no evidence for its claim at all, however many legs it has and however
independent they are of one another. Three witnesses that cannot separate H1 from H2 are not
three witnesses for H1; they are three witnesses for `H1 ∨ H2`, which is not what the name
being applied says.

Two things make this failure hard to see from inside, and both are worth designing against:

- **The evidence was recorded honestly and only the NAME overclaimed it.** Every one of the
  203 ledger rows gave its reason as "`& 1` flag test, RET 4 and a call to operator_delete" —
  a true statement that does not entail the name beside it. So the artifact was never lying,
  and no reader comparing the `reason` column against the evidence would find a discrepancy.
  **A row whose `reason` does not ENTAIL its `name` is a defect class worth a mechanical
  check**, and it is invisible to every check that grades reasons for truth.
- **"It keeps one vocabulary for one thing" is a reason to reuse a TOKEN, not evidence for the
  CLAIM the token makes.** The applying round gave exactly that as its justification, and it
  is a good answer to the half of the question it addresses (which of *our* names to use). It
  silently also answered the half it does not (which of two real things this *is*). Watch for
  a justification that is about your naming convention where the claim is about the binary.

**And the repair's guard must be TWO-SIDED, because a positive-only guard is the defect.** The
repairing apply asserted the array bit ABSENT on its targets — and FIRST asserted it PRESENT
on the 9 known-vector controls, refusing the whole run if the discriminator could not fire
where the array path had been read by hand. Prefer ground truth to constructed poison for
these arms: feeding the 9 genuine vectors to the "must not be a vector" rule and requiring all
9 rejected is exactly the arm the original probe never had, and it needs no fixture.

## A lesson you keep re-learning is a missing CHECK, not a missing note

Measured on a long-running project, by tagging its own lessons file — 192 blocks, 844 KB,
written faithfully round by round over months. How many blocks had to re-learn each theme:

| theme | blocks |
|---|---|
| something was stale (an artifact, a number, a note, an installed copy) | 59 |
| a count reported without the population it is a fraction of | 32 |
| a check that cannot fire, cannot fail, or grades nothing | 30 |
| a pattern, API or fact recalled instead of derived | 30 |
| a zero or a gap produced by the instrument, not the binary | 23 |
| the cascade reverting what was applied | 23 |
| our own applied work read back as independent evidence | 20 |

(Regexes over prose, overlapping, with 32% of blocks untagged — so every count is a floor.)

**These were all written down before they recurred.** Twice in one session, a lesson was
re-learned by the round *immediately after* the round that recorded it. That is the number
worth internalising: **writing a lesson down has a measurable failure rate**, and a lessons
file is the instrument that measures it.

So when a round ends with "we keep making this mistake", the productive question is not *how
do I write this more memorably* but **"what artifact join would have SEEN it?"** In this
project the things that actually stopped defects recurring were checks, not prose: a gate that
refused destructor names at vtable slot targets; a corpus query tool that *refuses* on a stale
stamp instead of documenting staleness.

**The worked example, because the shape generalises.** A round named 9 functions
`<Class>::vector_deleting_dtor` after reading their bodies. A later round applied the same token
to 203 more that were structurally different. Every one of the 212 rows was individually
well-evidenced and every gate stayed green for four months, because **no check compared rows
that SHARE A NAME against each other** — the defect lived in the vocabulary, where no per-row
check can reach. The join that finds it is small: *for every name token this project INVENTED
(the binary's own symbol table never spells it) that spans more than one round, require a
registered rule that all its rows satisfy.* On that binary the first clause gives 66 of 4678
tokens and the second cuts it to 10 — a reviewable population, not noise.

Three things made it a real gate rather than a gesture:

- **Validate it against the historical defect, not against a poison.** Replayed over the
  artifacts as they stood before the repair, it fires on exactly the offending row set. A poison
  you wrote to be caught proves far less.
- **Its population must depend only on committed inputs.** A draft also consulted a
  git-ignored export, which would have let the *scope of the check* drift with a cache. The
  weaker evidence moved into the ledger's `witness` column instead, where a reader can see it.
- **Join both directions and refuse on an empty population** — an unregistered token, a stale
  registration, a drifted count, and a population of zero all have to raise.

And when the recurring lesson genuinely is discoverability, prefer a tool that **recomputes and
writes nothing** to an index file. The same project's hand-written "current state" section sat
thirty-four versions behind the truth with accurate prose beside it; a derived query cannot go
stale, and a committed summary always can.

## A bound is not a discriminator — and passing the bound is not evidence about attribution

A containment check — *every recovered cell must lie inside the object's known size* — is a real
assertion. It fires on real data, it catches arithmetic that overruns, and it is worth writing.
It cannot catch a cell attributed to the **wrong, smaller** object, because such a cell is
comfortably inside the bound by construction.

Measured. A new witness followed constructor calls whose receiver resolved to `this + K` and
rebased the callee's cells by `K`. One body allocated a 0x134-byte block of its own, did
`LEA reg,[EAX+4]`, and built a four-element array inside *that* block: the receiver really was
`something + 4`, just not `this + 4`. The rebased cells reached offset 308 in a 1,124-byte class
and the containment check passed cleanly, as it always would — the misattributed object was
smaller than the one it was charged to.

What caught it was a human reading one promoted row that looked wrong (an offset of 4 in a class
whose base is 932 bytes), and that only happened because the rule **printed its promoted population
in full** instead of summarising it to a count. The eventual fix was a different question
altogether: not *"is this cell inside the object"* but *"is the tracker still on the object"* —
answered by counting how many times the tracker had been re-seeded onto an allocation result.

Two rules, and the second is the load-bearing one:

- **Ask what your check is a bound on, and what it is silent about.** A range check grades
  arithmetic. Attribution — *whose* object this is — is a separate claim and needs its own witness.
- **A rule that can promote can mis-promote, so print the promoted population in full.** With a few
  hundred rows the audit is free and happens by eye on every run, and it is the only thing that
  found this. A count would have said `360 sites` and been believed.

## A documented escape that requires hand-editing a constant is not implemented

A gate with a pinned census will, sooner or later, deadlock against the thing that produces its
input. The usual fix is an escape argument — a flag that downgrades one arm so the pipeline can
take one step and re-prime itself. Check that the escape can actually be *taken*.

Measured. A signature sweep pinned four censuses (A1–A4) and then joined against a probe's
artifact (A5). The probe built its population by reading the sweep's own output, so after an
apply that GREW the population neither could move first — documented, understood, and solved with
a `bootstrap` flag that downgraded A5. The flag was useless: **A4 raises before A5**, and a
bootstrap run abstains on every row *by construction*, so A4 failed against any pin describing
real verdicts. The only path through was: edit the pin to the abstain-everything value, run
bootstrap, re-prime the probe, run bare, edit the pin back.

That procedure works and is a trap. **A constant that must be edited back is one interrupted
session away from being committed as the gate** — after which it passes forever while grading a
state that only exists during a bootstrap.

The fix is not discipline, it is an assertion that knows which mode it is in: under the escape
flag, assert what the escape run *must* look like (here: every row abstains, checked against the
row count) rather than skipping the arm. That is strictly stronger than a skip and needs no edit.

**Test: can the documented recovery procedure be executed without modifying a checked-in
constant? If not, the procedure is a bug report about the gate.**

## Report the ROOT CAUSE of a dependency refusal, not the symptom

When a build has an ordering constraint, refusals get bucketed by the immediate blocker — "base
not built", "input missing", "dependency unresolved". That is true and nearly useless, because it
does not say whether the blocker is a *scheduling artefact* (it will build later, or on a second
pass) or a *hard block* (nothing can ever build it in its current state).

Measured. A struct-building wave refused 23 classes under `base_not_built:<X>`. Splitting the
bucket by *why* the base was unavailable — `base_unsized:<X>` (no decided size, so no prefix can
ever be flattened from it) versus `base_not_built:<X>` (buildable, just not yet) — turned 21 of
the 23 into a finding about three specific classes, one of which single-handedly blocked a
16-member family. The same 21 rows went from a wall to a backlog item with three names on it.

The refusal bucket is a report to a human who has to decide what to do next. **Bucket by the
cause they can act on, not by the check that happened to fire.**
