# Trust model — the deep end

*Reference for the `ghidra-iterative-re` skill. Only `SKILL.md` is loaded when the
skill is invoked; this file is read on demand when its trigger fires. New lessons of
this kind belong here, not in `SKILL.md`.*

**Read this when:** you are deciding whether a piece of evidence is independent of your
own work: reading types back, auditing a tier, or wondering whether a name you are about
to trust came from the binary or from a previous round.

**In this file:**

- Laundering paths, and what the tier does not tell you
- Exploiting the filter rather than fearing it
- The TYPE axis: `DataType` has no `SourceType` at all
- `SourceType` is necessary but not sufficient

### Laundering paths, and what the tier does not tell you

**Two laundering paths promote `AI` markup out of the tier you filter on**, and your ledger
cannot see either, because the laundered rows are no longer tagged `AI`:

- **XML symbol import defaults a missing `SOURCE_TYPE` attribute to `USER_DEFINED`.** Any
  export/re-import round trip that drops the attribute promotes every agent name to the
  *highest* tier.
- **Version Tracking copies the SOURCE program's `SourceType` into the destination.** Agent
  names cross a build boundary still tagged `AI` and reappear as apparent second-binary
  corroboration — a self-harvest with an extra binary in the loop.

Audit both before using them. *(Both reported by an external review of this document and
not re-verified here; check against your install before relying on the exact mechanism —
but treat the hazard as real, because the failure is silent in the one direction your
ledger is blind to.)*

**Do not assume demangled names are `IMPORTED`.** Measured on a PE with 981 mangled
symbols: only **6** of 2640 functions carried `IMPORTED`, while **985** carried `ANALYSIS`.
The raw export label is importer-created ground truth; the *demangled name applied to the
function* is analyzer output one inference removed from it. Census your program before
building a trust rule on a tier.

### Exploiting the filter rather than fearing it

Four corollaries measured later on the same project, which make this cheap to *exploit*
rather than merely to fear:

- **Applying the HIGHEST-FAN-IN name in the binary is also an audit of every producer's
  filter, and it is the cheapest one available.** Measured: naming a custom arena allocator
  and its free — 852 call sites between them — made exactly one committed artifact change,
  in exactly one cell, because one sweep recorded the allocator's *name* beside the size it
  had measured and had no `SourceType.AI` exclusion. A wide net finds the holes; schedule
  the high-fan-in names EARLY for this reason, not only for the decompilation payoff.
- **The witness being unaffected is WHY the leak survives, not a reason to leave it.** That
  size bound came from a pushed immediate and a vftable store; the leaked name was
  provenance text sitting beside them, so nothing was mismeasured and nobody had cause to
  look. A committed artifact must not change because of what you chose to name last week,
  whatever the cell is *for*.
- **Put the filter's assertion BEFORE the write**, and **keep the ADDRESS beside every
  emitted name.** The first makes a failing filter produce no artifact at all instead of a
  plausible one that must be reverted by name afterwards — the "a run that RAISED must not
  have its output promoted" rule solved at the derivation rather than in the cleanup. The
  second is required because the ledger join is by address, and a producer must not recover
  the address from the name it just wrote.
- **Audit the whole producer, not the cell that moved.** The stability harness can only
  surface the leak whose target this round happened to name. Of that sweep's five name
  reads one was the defect and four were already safe — but the four had to be *read* to
  know it. A one-cell diff is a prompt to audit, not the scope of the repair.

### The TYPE axis: `DataType` has no `SourceType` at all

The trust model in `SKILL.md` is about *symbols*. **Types have no provenance field
whatsoever** — no tier to filter on, no `getSource()`, nothing marking a struct you
grew as yours. So the anti-circularity rule has no mechanism on the type axis, and a
harvester that reads a type name back is unguarded by construction.

Define it from the program's OWN record instead of from your notes. **A
version-controlled Ghidra program carries a complete type history**, and it is
directly readable:

```python
from java.lang import Object as JavaObject
df   = currentProgram.getDomainFile()
hist = df.getVersionHistory()                                  # number, user, comment
old  = df.getReadOnlyDomainObject(JavaObject(), n, monitor)    # open version n
...
old.release(consumer)
```

Diff the `DataTypeManager`s across each version boundary and you have a ledger of
every type you ever applied, attributed to the round that applied it — for free, if
your checkin comments name the rounds. Two mechanics that cost real time:

- **The consumer must be a `java.lang.Object`.** A plain Python `object()` makes
  JPype report *"No matching overloads found"* against a signature it obviously
  matches.
- **Diff per TYPE structurally, never per category or per count.** Measured on one
  project: the `/Demangler` category went **152 → 153 types across its entire
  history**, while 15 placeholder structs grew from 1 byte to full class layouts.
  A count view measures that whole population as `+1`. Fingerprint
  `(path, length, [(offset, component type name, field name)])`, or a component
  RETYPE or RENAME at constant size — exactly what a field-type upgrade is — is
  invisible.

**Your NAME applies create types.** Applying a qualified class name makes Ghidra
materialise a **1-byte empty placeholder struct** of that name, and the decompiler
will then happily type a `this` parameter with it. Measured: 30 such structs across
three rounds that applied only names. They are yours, they are circular to read
back, and nothing in a name ledger mentions them.

**And "a type we applied" is not one thing.** The tiers decide whether reading one
back is self-reference at all:

| tier | what it is | circular? |
|---|---|---|
| transcribed | you typed in a real vendor header (DirectInput, Win32, an SDK) | **No** — the authority is external. The `IMPORTED` analogue |
| derived | you derived it from this binary | **Yes** — the `AI` analogue |
| side effect | Ghidra materialised it because you applied a name | **Yes** |

**And `transcribed` needs a second column your notes probably do not have: WHERE FROM,
and UNDER WHAT LICENCE.** The tier records that a type came from outside; for any project
whose destination is a reimplementation, the property that actually matters is whether the
outside source can be carried into your source tree at all.

Measured: a vtable layout for a Microsoft COM interface was transcribed from a public
copy of the vendor header, and the licence was checked only afterwards — **LGPL-2.1**,
derived from Wine and MinGW-w64. Copyleft, in a repo aiming at a clean reimplementation.

The resolution generalises, and it is the same discipline as everything else here:

- **Treat the external layout as a HYPOTHESIS and verify it against the binary.** Here the
  argument count the program passes at each of its own call sites had to equal the
  parameter count the header declared: 11 of 11 agreed, and the semantics corroborated
  separately (the `Open` slot taking one flag value to host and another to join; a
  get-modify-set pair on the session descriptor matching two log strings). Those eleven
  are established by *measurement*, and the header is not their authority.
- **Keep only what the binary establishes; replace the rest with numbered placeholders.**
  The other 42 slots were unverifiable — nothing in the program calls them — so they
  became `slot_NN`. The compile check still passed identically afterwards, which is the
  proof the names carried no information: **removing them cost nothing and removed the
  obligation.** If a name is not verifiable against your binary, it is contributing
  licence risk in exchange for nothing.
- **Audit the headers you already have.** The same sweep found one sibling header with its
  provenance recorded and self-derived (parameter counts taken from `__stdcall` name
  decoration in the import table — no external source at all), and another with *no
  provenance statement whatsoever*, silently carrying the same unanswered question.

Two adjacent traps found in the same file, both worth copying:

- **An MSVC-ism makes a header uncompilable off Windows, so nobody ever compiles it.**
  `__stdcall` is a keyword gcc rejects outright. A header full of offset-critical layout
  had therefore never been compiled once since it was written — the dormant-defect shape
  again. Add a portability shim (`__attribute__((stdcall))` on x86, empty elsewhere) so
  the layout can actually be asserted.
- **Verify the offsets with a COMPILER, not with your own arithmetic.** Re-deriving struct
  offsets in Python is the project's arithmetic checking itself; `offsetof` in a compiled
  translation unit is an independent evaluator. And poison it — move one slot by four and
  require the compile to fail, per assertion.

Collapsing these makes the guard fire on every Win32 callback in the binary and
reads as a catastrophic exposure. Separating them is the difference between a number
you can act on and one you will learn to ignore.

**Finally, ask whether the binary already explained the reference.** A demangled
signature reading `void Dwim(CMessage * this, …)` gets `CMessage *` from
`?Dwim@CMessage@@UAEX…`, which predates every apply you made — you changed what
`CMessage` is *defined* as, not the reference to it. Of 1409 rows carrying a ledgered
type on one project, **750 were explained by the mangled name and were ground truth;
659 were not, and only those were exposure.** Same shape as the `SourceType` filter:
the question is never "does our name appear", it is "would it be here if we had done
nothing".

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

### What an apply costs the witness that justified it

- **THE WITNESS THAT MEASURED A SIZE STOPS BEING EVIDENCE THE MOMENT THAT SIZE IS APPLIED.**
  Measured: a probe established a class's size by following a pointer and recording the furthest
  byte the code reached. Applying a struct of exactly that size made the same probe return the
  same number **because the applied type now bounds what the decompiler will report** — the
  answer became a consequence of the apply rather than of the program. Nothing about the probe
  changed, and its output looked like an independent confirmation. Two rules follow:
  - **Re-running a probe after acting on it produces a CONFIRMATION, not a measurement.** Record
    that distinction in the artifact itself, in a column, naming which run was the measurement.
    A future reader has no other way to tell, and "the probe agrees" is the most persuasive
    wrong sentence available.
  - **This is the self-harvest trap on the TYPE axis**, where there is no `SourceType` to filter
    on. The symbol filter that stops you reading your own names back has no equivalent here; the
    only defence is the recorded provenance of the number.
- **A PRE-APPLY ARTIFACT IS AN ORACLE THE POST-APPLY RUN CANNOT INFLUENCE — USE IT WHILE IT
  EXISTS.** The committed copy of a witness file, taken before the mutation, is the one piece of
  evidence the apply provably did not touch. Diff against it *keyed on identity*, not
  positionally: row counts and ordering both move, and a positional diff on a grown file reports
  most rows as changed and hides the few that actually are. Measured, that turned a 387-of-566
  "everything moved" into the true answer — **518 rows in common, of which 8 differed, all in one
  column.** The pre-apply copy is also what lets you state a payoff against a *structural zero*:
  names that did not exist in the struct before the apply cannot have appeared in any earlier
  decompilation, so every occurrence is necessarily new and a post-apply-only count is a real
  delta rather than the "before == after by construction" trap.
- **THE OUTCOME TO WANT FROM A RE-APPLY IS MORE EVIDENCE AND THE SAME DECISION — STATE BOTH
  HALVES.** Measured: the raw evidence file grew 380 → 502 rows with **zero rows removed**, while
  the adjudicated layout it feeds stayed byte-identical. Growth alone could be noise; stability
  alone could be a sweep that failed to see anything new. Together they are corroboration, and
  neither number means much reported without the other.

### A provenance boolean is almost always too few tiers — and a claim written INTO the program is read by someone who cannot re-derive it

**Measured.** A round was about to stamp a comment into the program for each of 101 classes,
recording whether its base class's name came from the binary. The first cut carried
`base_named = not name.startswith("UNKNOWN_")`. Checked properly, the 17 bases fall into **four**
tiers, not two:

| tier | n | what it means |
|---|---|---|
| exported mangled vftable symbol | 4 | the binary states the name outright — ground truth |
| the program's own registry name string | 100 (of the classes) | the binary states it, at a weaker tier |
| the project's adjudicated inference | 5 | **not stated by the binary at all** |
| none | 8 | no name recovered; the placeholder is an address |

The boolean would have written a comment calling an *inferred* class name a name from the binary.
That is the `SourceType` laundering the trust model exists to prevent — committed in prose instead
of in the symbol table, where no filter can catch it.

Two riders that generalise past this instance:

- **An annotation written into the program is the one artifact whose reader cannot re-derive it.**
  A CSV row sits beside its producer and its evidence columns; a comment in a disassembler is read
  months later by someone with no way to check where the claim came from. So the provenance goes
  *in the comment text*, at the tier's real name, not compressed to "named".
- **Any column that answers "where did this come from" should be an enum with a documented meaning
  per value, and the producer should RAISE on a value it cannot classify** rather than defaulting.
  A default is how a fifth tier gets silently absorbed into one of the four.

### Find the corroborating constant by COUNTING, not by knowing it

**Measured.** A naming round needed a third, independent witness that a set of 224 bodies were
deleting destructors. The natural check is "does it call `operator delete`" — and hardcoding that
address would have made the third leg a restatement of the first, since the address was known only
because of earlier work on the same hierarchy.

Instead the probe **counted callees across all 224 bodies and reported whichever dominated**,
refusing (raising) if no callee reached even half the population. One address came back at
**224 of 224**. Only afterwards was it looked up — and the binary's own export table names it
`operator_delete`.

That ordering converts the weakest leg into the strongest one. A discovered constant that ground
truth then confirms is evidence; the same constant typed in from the start is an assumption wearing
a check's clothing. **Whenever a check needs a magic address, ask whether the data can be made to
volunteer it, and make "nothing dominates" a refusal rather than a fallback.**

### An artifact changing is not automatically a leak — read the DIRECTION, and read the consumers

Two adjacent lessons from one round, because the instinct they correct is the same one.

**Direction.** A committed evidence artifact changed on **exactly** the addresses a naming round had
just written to — the precise signature of an AI-name leak. It was the opposite: the values went
`FUN_xxxx → ""`, the anti-circularity filter dropping the round's own names *out* of evidence,
which is the filter working. Had the direction been `"" → <our name>` it would have been the defect.
**The count of changed rows tells you nothing; the before-and-after values tell you everything.**

**Consumers.** In the same round a second artifact's `signature` column genuinely did start carrying
values derived from the round's own markup, and that column is not name-filtered. Rather than rank
it as a finding, its two real consumers were opened first: one keys on an unrelated enum name, the
other covers a disjoint population — **overlap with the affected rows: 0 in both**. Severity is a
property of what a column FEEDS, not of how contaminated it looks. This is the same rule as
"ask what each occurrence feeds before asking how wrong it is", and it is worth restating here
because a contaminated-evidence finding is *satisfying*, which is precisely when it goes unchecked.

### Before discarding a circular witness, try restating it WITHOUT the name

A witness rejected as circular is often circular only in the way it was *phrased*. The underlying
fact frequently involves no name at all, and rewriting it costs one line.

**Measured.** Naming a class's scalar destructor, the obvious corroboration was *"`X::vector_deleting_dtor`
calls this body"* — and it is genuinely circular, because an earlier round applied that name. But the
name is doing no work in the claim. The same structural fact stated positionally is
*"the target of **slot 38** of `X`'s vtable calls this body"*: the slot index is ground truth from an
earlier ABI finding, the pointer is read out of **memory**, and the call edge is read out of the
instruction stream. **No symbol is consulted at any point**, so the witness cannot be satisfied by
the project's own markup — and it is the same evidence, minus the laundering.

The general move: write the candidate witness as a sentence, then delete every proper noun. If what
remains still identifies the thing — an address, an ordinal, a slot, an offset, a byte pattern —
the witness was never circular, only badly worded. If nothing remains, it really was the name doing
the work, and it should be dropped.

**Budget for a sibling witness that survives inlining.** In the same round, four of six targets could
cite a call to an exported base constructor/destructor; the other two could not, because the compiler
had **inlined** the base destructor and there was no call left to cite. A witness kind that depends
on the compiler not having inlined something will silently cover only part of any population, and
the gap is not random — it correlates with small, hot, frequently-called bodies, which is exactly
where the high-fan-in naming targets live.

### The decompiler renders YOUR member names, and that is where self-harvest is least visible

The previous item is about phrasing a witness. This one is about where the bad phrasing comes from,
because once a struct is applied the decompiler *hands* it to you.

Apply a struct with a member named `Position` at `+0xc`, and every body touching that offset now
decompiles as `this->Position`. Read three such bodies and the impression is overwhelming that the
program agrees with the name — but the text was generated **from your own markup**, and it would
read identically if the name were wrong. A decompiled body is the place self-harvest looks most
like corroboration, precisely because the corroborating sentence writes itself.

**The mechanical discipline: cite the OFFSET and the operations, never the rendered member name.**
"`AboveGround` uses `this->Position`" records nothing. "`AboveGround` loads the dword at `[this+0xc]`,
null-checks it, and uses the pointee's fourth field as the index into an already-decided
`LayerGrid`" is a claim that survives renaming the member to `m_0xc` — which is the test. If a
witness sentence changes truth value when you rename the member, it was never evidence.

**The corollary bites the instrument, not just the prose.** A very common field-locating route is to
grep decompiled C for the decompiler's placeholder spelling of an unnamed member (`m_0x1c4`,
`field_0x1c4`, `unk_1c4`, depending on the setup). That route is **structurally blind to every cell
you have already named** — and "cells already named, on evidence somebody later doubted" is exactly
the population a re-examination round is about. Measured: three field names were re-examined and the
established text-level locator would have returned a confident zero for all three. Scanning
instruction operands for `[reg + disp]` sees them regardless of what the member is called and
regardless of whether the body's `this` is typed.

So: **before reusing a text-level member locator, ask whether the cells in question are named.** If
they are, the tool's silence is a property of the tool.

## The lower tier is for a strong ARGUMENT with a weak CITATION — they are different things

A two-tier confidence scheme (`decided` / `held`, or any equivalent) is usually explained as
"confident" versus "unsure". That framing is what erodes it, because a reader with a genuinely
convincing case will reach for a weaker citation to clear the bar rather than record the case
honestly at the lower tier.

The distinction that actually holds up: **`decided` is about what a MACHINE can re-check; the lower
tier is where a strong human argument goes when no machine-checkable witness exists.**

**Measured.** A function was identified as the writer half of a save/load pair on an excellent
argument — it walks the same two globals as the already-`decided` reader, and writes exactly the
five values per entry that the reader consumes, with the same terminator. It had no citable witness:
no string of its own, and its only caller carried the PROJECT's own name rather than a ground-truth
one, so the "called by a ground-truth name" witness was unavailable and the weak "references this
global" witness never decides by policy. Recorded at the lower tier with the argument in the
rationale, and the ledger stayed honest.

- **Never strengthen a citation to match your confidence.** If the argument is good and the witness
  is not, that is exactly the state the lower tier exists to record.
- **Symmetry with a decided sibling is an ARGUMENT, not a witness.** It is often right, and it is
  never something a probe can re-check at an address.

## When a name encodes a claim about SAMENESS, verify it at the byte level

Naming several functions `Foo`, `Foo2`, `Foo3` asserts that they are the same routine emitted more
than once. That is a real claim about the binary and it is cheap to check: **compare the raw bytes.**

**Measured.** Three candidate duplicates of a small checked-read helper were compared 50 bytes at a
time; all three were identical to the original apart from the two relative `CALL` displacements,
which necessarily differ by position. That turned "these look like duplicates" into "these ARE
duplicates, and the numeral records emission per translation unit rather than any difference in
behaviour" — a sentence that can go in the rationale and be checked by anyone later.

Without the check, a numeric suffix silently asserts sameness on the strength of a decompiler
listing looking similar, which is exactly the kind of unrecorded inference the tiering exists to
prevent.

## An APPLIED `provisional` is treated as fact, because the decompiler does not render a tier

Confidence columns are read by people and by adjudication scripts. They are not read by the
decompiler, by a struct listing, or by the next round's author opening a body. **The moment a
provisional layout is applied to the program, every consumer sees a layout.**

Measured (1999 MSVC/x86 game). A 12-byte record was inferred early on from **one** witness kind,
recorded honestly as `provisional` with `witness_kind_count == 1`, and applied. It is 8 bytes. It
stood for **124 program versions** — through dozens of rounds, several of which worked in the very
subsystem that contains it — and was refuted, when someone finally looked, by **three separate
exported functions**, none of them obscure: one indexed the records at `i*8`, one allocated `n*8`,
and one asked for 40 records and received 320 bytes.

The tier was correct and changed nothing. Nobody was ever prompted to revisit it, because a
provisional row that has been applied looks exactly like a decided one from every direction that
matters.

- **A provisional row that is APPLIED needs a standing route back to it**: a backlog line naming it,
  a probe that re-derives it, or a periodic sweep of the low-witness rows. A tier with no mechanism
  behind it is a label, not a plan.
- **The cheap first pass is a query, not a project.** "Which applied rows have
  `witness_kind_count == 1`?" is one filter over the layout artifact, and it ranks the whole
  backlog by exactly the property that produced this defect.
- **Prefer NOT applying a one-witness layout to applying it with a caveat.** An unapplied inference
  costs a round its decompilation improvement; an applied wrong one costs every later round its
  premises, and is much harder to notice because the improvement it bought is real.

> Corollary for the gate: when the correction lands, the drift detector that compares program
> against artifact WILL fire, and that is the gate working. Move the artifact row. Do not relabel
> the row `confirmed` to satisfy a check that defines `confirmed` in terms of an evidence file the
> new witness does not live in — that is forging a witness kind to pass your own gate.

## Check who named the thing your citation cites

A citation kind that says *"this body calls `fwrite`"* sounds like it rests on the C runtime. It
rests on whoever decided that function is `fwrite`. If that was you, the kind is circular — and the
circularity is one hop long and completely invisible in the citation string.

Measured. A project implemented a `save_record` witness kind for naming `Save`/`Load` bodies: assert
the body calls a stdio stream primitive, matched by the callee's name. The name filter that excludes
the project's own AI-tagged markup answered `FUN_004c4f2d` — because `CRT::_fwrite` was a name **that
project had applied itself**, 90 program versions earlier, from its own reading of the argument
shape. The kind's entire purpose was to be independent of project markup, and every row using it
would have rested on project markup, permanently.

Two things caught it, and both are worth copying:

- **The checker asked through the AI-excluding name accessor, not through `getName()`.** A
  string comparison against `callee.getName()` would have passed cleanly. The filter is only
  protective if the checker actually routes through it.
- **The batch was pre-flighted before the apply.** The catalogue had a mode that runs every
  checker against candidate rows that are not yet in the program, precisely so a bad citation costs
  a re-edit instead of a program rollback. It refused four rows and named the reason.

**The repair is to ground the claim in something the FILE states**, not something you decided: a PE
import, an export mangling, a relocation, a byte pattern. Here the fix was a call-graph walk —
measured at exactly **2 hops** from the write primitive to `KERNEL32!WriteFile` and from the read
primitive to `KERNEL32!ReadFile` — so the citation became `save_record:<callsite>:<import>` and no
project-chosen name appears at any link. The walk is bounded by a pinned constant, because an
unbounded one reaches everything.

> Generalise it: for every witness kind, write down the chain from the citation to something in the
> file, and name the tier of each link. A chain that passes through your own symbol table at any
> point is not a witness, however many links it has.

## A tier scale that runs "strong" to "weak" rounds WRONG up to WEAK

Provenance tiers answer *how well is this supported*. They do not answer *is this true*, and most
tier vocabularies are built without noticing the difference — because they are designed during the
phase when everything recorded is presumed right and only the strength of the support varies.

**Measured.** A curated artifact of recovered field names had two tiers: `decided` (a citable witness)
and a demoted tier meaning *present in the program, evidence judged insufficient by a recorded round,
kept so a rebuild cannot destroy it*. Both were applied to the program and both reached the emitted C
header, on the stated grounds that "both describe what the program holds." A later round then
**refuted** two of the demoted names — measured them onto cells whose contents the noun does not
describe (one was a reference count, one a one-shot latch). There was no value to put in the column.
Filed under the demoted tier they were indistinguishable from weakly-witnessed-but-probably-right, and
the emitted struct ended up carrying two members that claimed to be the same thing.

**Add a REFUTED value, and keep the row.** Deleting it loses the measurement that condemned the name,
and invites the next round to re-derive the same wrong name from the same evidence — which, for a name
derived by a mechanical rule from an accessor, it certainly will. The row is now the record of a
finding rather than a claim.

Two consequences worth taking:

- **Refuted and demoted must be treated differently by consumers.** A demoted name is one nobody has
  disproved; withholding it discards work. A refuted name is one somebody has. Only the second should
  be withheld from an emitted artifact, and conflating them either leaks wrong names or silently drops
  good ones.
- **A skip must be PRINTED by every consumer that performs it.** A refusal that produces no output is
  how the name comes back: the next person sees a row in the artifact, no warning anywhere in the
  build, and "fixes" the omission.

**And check what the consumers actually read.** In the measured case, the header emitter had *never
looked at the tier column at all* — it had been overlaying every row regardless of tier since the
column was introduced. The tier was doing real work in one consumer and none in another, and nothing
said so. When you add a tier value, grep every reader for the column; a tier only exists where
something branches on it.

## The tier of a row and the availability of evidence are different questions

A findings file's confidence column describes how sure the analyst was. It says nothing about
whether a machine-checkable witness exists for the claim — and the second is what a later round
needs.

Measured: an investigation proposed ~40 function names and marked them all *strong*. Priced against
the program before the applier was written: **42 candidates, 27 with any citable witness at all, and
of those 27 a further 12 whose only witness was "this body calls a generic helper"** — a fact true of
hundreds of bodies that discriminates nothing. Fifteen names were applied; the reasoning behind the
other twenty-seven was probably fine, and it is not evidence.

- **Run the pricing as a separate read-only artifact, FIRST**, and keep its output. It decides the
  round's scope, and the next session starts from it instead of re-deriving it.
- **The test for a witness is counterfactual: what would it look like if the proposed name were
  WRONG?** If it would look identical, it is not a witness for that name.
- **Two witness KINDS are not two witnesses unless the second discriminates.** Four sibling
  functions each citing "an identically-named export calls me" plus "I install a vtable" is strong
  only because each installs a *different* vtable. Had they shared one, the count would satisfy the
  two-kind bar while establishing nothing about which sibling this is. A kind count cannot check
  that; the author owes it.


## An anti-circularity filter can fail in TWO directions, and only one of them usually has a guard

You build a filter so your own applied names and types cannot come back out of a harvester as
evidence. You then write an assertion proving the filter worked: *no name we applied appears in
a committed cell*. That assertion is real and it is worth having. It watches exactly one of the
two ways the filter can be wrong.

The other way is that the filter **suppresses evidence that was never yours** — and nothing in
the design notices, because every check is looking for contamination arriving, not for ground
truth departing.

Measured on the 1999 MSVC/x86 project. The suppression set was built from a ledger of "types we
changed", and 32 of its 362 names were classes the COMPILER had mangled into the binary's own
export table — the demangler created them at import, before any apply. The filter had been
writing `undefined` over them in the committed signature column of the file twenty consumers
read: **711 ground-truth type references in 626 rows**, three quarters of everything it
suppressed. The assertion beside it was green the whole time and correctly so — nothing had
leaked *in*.

Two things make this specific rather than a warning:

- **The ledger answered a different question than the filter asked.** The `ours` verdict was
  built from *did we change this type's DEFINITION* (which version boundary, which artifact
  claims the path). The filter needed *would this NAME be in a type position if we had done
  nothing*. For a demangler-created category the second answer is always **yes**, however much
  of the struct you later filled in. **A provenance verdict is only usable by a filter that asks
  the same question the verdict answers** — check that explicitly, because the two questions
  read alike and the column name (`ours`) suits both.
- **The correct rule already existed and was evaluated over too small a domain.** The filter had
  an exemption — *"a type explained by the binary's own mangled name is ground truth"* — asked
  **per address**: does the mangled label at THIS function spell the type? But whether `CGobject`
  is a word the binary uses is a fact about the **whole program**. A correct rule scoped to the
  wrong domain looks exactly like a correct rule, and it fails only at the sites the domain
  excludes — which are, by construction, the ones nobody is looking at.

**So: when you add a suppression rule, state what a false POSITIVE costs and instrument that
too.** The cheap version is a count — how many names the rule suppresses, and how many of those
the binary itself supplies — printed on every run beside the leak assertion. A filter that
reports only what it caught cannot tell you what it destroyed.

**And when you fix such a rule, run the OLD rule against TODAY's program first.** The committed
artifact is not the old rule's output unless nothing else has moved since it was written, which
is the very thing in question; a control run separates "my rule change did this" from "the
program drifted" from "my new code has a bug", and nothing about staring at the diff will. On the
project above the control reproduced three artifacts byte-identically, which is what made the
626-row diff attributable at all.

**Then check every OTHER producer of the artifact the filter protects.** The same project found a
second writer of that file — the original, from an early phase, still named in the replay
sequence as *"the only step that writes it"* — which applied **neither** filter and carried
**neither** assertion. It had not been edited in months, so nobody had re-read it; a literal
replay would have regenerated the artifact fully contaminated and it would have looked entirely
normal. A guard is worth what its weakest writer applies, and the producer that never changes is
the one that never gets re-read.

## Progress elsewhere can quietly degrade a probe keyed on a shared attribute

A probe that identifies candidates by a *shared attribute* — most often size — gets weaker every
time the project creates another thing with that attribute. Each individual step is correct, no
gate fires, and the probe's discrimination erodes anyway.

Measured. A block-copy probe lists the known types matching each copy's byte count, so an analyst
can pick the likely element type. Giving one previously-unmodelled class a real struct made it a
third 64-byte type, and five rows went from two candidates to three. Nothing about the copies
changed; the probe just answers a slightly less useful question than it did the day before.

This is not a defect to fix and usually not a reason to act — but it is a real cost, and it is
invisible unless someone writes it down, because *the artifact diff looks like noise and the
change is genuinely correct*. Record it with the round that caused it. A probe whose candidate
lists have been growing for twenty rounds is one worth re-examining, and nothing else will ever
prompt that.

**The general shape: an inference that ranks candidates by a shared property has a denominator
that your own work keeps increasing.** Watch it the way you would watch a false-positive rate.

### A finding with no channel: widen the producer, never hand-write the answer

A recovered fact sometimes fits no artifact you have. Measured: a cell needed a NAMELESS type
correction (`dword` -> `float`, evidenced by a float read added to a float constant), and of the
three candidate homes, one was hand-curated but required a name on every row, one had exactly the
right schema but was regenerated by a gated sweep, and the correct applier read a third artifact
whose types came from a witness that structurally cannot tell a float from an int.

The two shortcuts are **invent a name so the row fits its artifact's shape**, and **hand-edit the
regenerated file**. Both produce a green tree and a false record — the first launders a
description into a recovered symbol, the second is overwritten on the next pass or, worse,
survives as a row no producer stands behind.

**Widen the PRODUCER's population instead and let its own rule decide.** That grades the claim by
a standard the finder did not write, which is the whole point of the trust model: if the sweep's
evidence bar then refuses the cell, the refusal is information about the evidence rather than a
verdict on the finding, and it is recorded where the next round will look. Cost is one literal in
a sweep; the alternative costs the artifact's meaning.

### Naming what the binary does not name: the provenance needs a GUARD, not a convention

A project can reasonably decide to name fields and functions the binary never spells — a
recovered mechanism is worth a label even when no symbol supplies one. The danger is not the
naming; it is that **an invented name reads exactly like a recovered one six months later**, and
the caveat that distinguished them lives in prose that nobody re-reads. (Measured elsewhere in
this file's project: three "no evidence" verdicts hardened into claims about the game within
days, because the scope caveat sat one sentence away from the conclusion.)

So enforce the declaration mechanically, and check it **both ways**:

- a row claiming **invention** must say so in a fixed marker AND cite **at least two distinct
  addresses**. An invention still has to rest on recovered mechanism; without that floor you have
  re-admitted the plausible-name-resting-on-a-zero-init failure the witness rule exists to stop;
- a row claiming a **binary source** must cite a **mangled symbol**, so "the binary spells this"
  is checkable rather than asserted.

**Put the marker in the PROGRAM as well as the artifact.** A field comment or a symbol comment
that opens *"NAME APPLIED BY ANALYSIS — NOT FROM THE BINARY"* travels with the thing it labels,
and if another applier already treats a comment as a prior decision it may not silently overwrite,
the marker becomes a guard rail rather than a label. An artifact column alone is lost the moment
somebody reads the struct in the GUI.

**The guard will catch your own census before it catches any poison, and that is the better
demonstration.** Measured: the first apply attempt was refused because one real citation opened
with a parenthesis instead of the exact marker. Write the guard so the genuine rows must satisfy
it, then poison it.

### A queued item's stated BLOCKER is an untested premise, and it decays faster than its evidence

A premise recorded in a notes file is a claim with a provenance tier like any other — but a
**blocker** decays differently from the evidence beside it, and worse. Measured across five queued
items in one project: **four had the wrong blocker**, and in every case the *evidence* had survived
while the reason-it-could-not-be-done had not.

The four failure shapes are worth recognising by name:

- **The blocker names a missing channel that has existed for rounds.** *"There is no channel for a
  nameless type correction"* — the upgrades artifact is exactly that channel and already held five
  rows for the very class.
- **The blocker is true of a tool and irrelevant to the route.** *"The apply script excludes
  width-changing retypes"* — true of that script, which is a drift-repair tool for 4-byte scalars
  and not the path a layout decision travels.
- **The blocker is true and means the opposite of what it looks like.** *"Fails at HEAD"* was
  accurate and was the finding, not the obstacle.
- **The blocker names a real hazard that is not the dominant one.** See the vptr-store case in
  `harvesting-traps.md`: the named hazard explained 1 of 21 orphans.

**Price the BLOCKER before pricing the round; it costs one artifact read.** A stale blocker is more
expensive than a stale finding, because a wrong finding gets refuted the moment somebody works on
it, and a wrong blocker guarantees nobody ever does. In the same way an unrun sweep's *"unknown"*
reads as an opportunity forever, a decayed blocker reads as a closed door forever — and neither
carries a timestamp saying when it was last true.

### The AGREEMENT CENSUS is what admits a new witness kind — and its disagreements are the finding

A new witness kind arrives with no track record, and the temptation is to justify it by argument
(*"a member function writing `this+X` is obviously a field"*) and then measure only what it adds.
That gets the order wrong. **Grade it first against everything the project already decided by
other means**, and state the result as a fraction of the cells it graded, not as a count.

The bands worth separating, given committed rows from the other channels:

| band | meaning |
|---|---|
| `exact` | same offset, same width as a committed row |
| `subfield` | wholly inside one committed row — a narrower access into a known field |
| `new` | no committed row overlaps it: the payoff |
| `widen` | same offset, wider than the committed row |
| **`straddle`** | one access crossing a boundary **between two committed rows** |

Measured on a channel admitted this way: 1250 cells over 90 already-decided classes — `exact` 32%,
`subfield` 52%, `new` 15%, `widen` 0, `straddle` **1**. **85% of what the new instrument said about
already-decided classes was something a different instrument had already said**, which is what
licensed believing it about the rest.

**And the one straddle was a real defect in the committed artifact, not in the new channel.** A
cell held as two 4-byte integers was read by the program as a single 8-byte float; the constructor
had zeroed it with two dword stores, which is exactly why the initialisation-based producer could
not tell the difference and why the new channel could. That is the shape to expect: a new witness
kind's disagreements cluster precisely where the old kinds are structurally blind, so **grade the
disagreements one at a time before assuming either side is wrong.**

Two rules fall out:

- **Pin the straddle count.** It is the running measure of how much this channel contradicts the
  rest of the project, and it must be adjudicated when it moves, never re-pinned.
- **Do not repair the other producer's artifact from inside the new one.** Record the correction
  with its addresses and let the round that owns that producer make it, with its own census and
  approval. A channel that both proposes rows and silently rewrites its calibration source has
  stopped being independent of the thing it is being graded against.

### The anti-circularity name filter is correct for EVIDENCE and wrong as a measure of WORK

The rule at the top of this file — a harvester must exclude the names your own project applied —
is usually implemented as one helper: *"give me this function's name, treating anything we applied
as no name at all."* That helper is right, and it is the reason the trust model holds at scale.

**Then someone prints its result under the heading `named` and reads the complement as a to-do
list.** Those are different questions, and the gap between them can be enormous. Measured: a
small-body shape census reported six families as *"wholly unnamed … the naming rounds waiting"*,
totalling 1,266 functions. Of those, **1,174 (92.7%) already carried a name the project had
applied** and **92 (7.3%) carried none at all**; five of the six families were at 100% already
named. The headline item — *"769 of 769, one naming rule away from 769 names"* — was worth **zero**.
The entry sat in the project's own backlog, quoting a producer that was faithfully answering a
different question, and it **overstated the available work 13.8x**.

**The fix is three columns, not a better sentence.** Any census that reports naming coverage should
separate:

| column | meaning | is it work? |
|---|---|---|
| `binary` | the binary itself names it (a mangled export, a debug symbol) | no — and it is usable as evidence |
| `ours` | this project applied the name; it is the AI tier, excluded from evidence BY DESIGN | **no** |
| `nameless` | no name from any source | **yes** — this is the naming round |

And derive `ours` from the **ledger by address**, not from the program: a symbol's provenance tier
is exactly what the evidence helper already consumed, so re-deriving it from the tier reproduces
the conflation.

### A population that exceeds its own denominator is not a subtle signal

The same incident had a one-line tell that sat unread for a fortnight. The project's
"functions with no name" denominator was **946**. The two largest "wholly unnamed" families alone
were **971**.

A subset cannot be larger than the set. When two numbers about the same population cannot both be
true, that is not a rounding disagreement to note and move past — it means **the two numbers answer
different questions**, and finding out which is usually one read of each producer's predicate. Here
it took two reads and refuted a queued round.

Build the comparison in where you can: a census that reports a subpopulation of a tracked
denominator should assert it is no larger than that denominator, and name both producers in the
message. A check that reads *"family total 971 exceeds D1 946 — these two populations are not the
same population"* turns a fortnight into a run.

### A local override is an undocumented bug report — and it carries a date

When you confirm a defect in a shared library, grep for the parameter that disables it before you
write anything. Measured: a scanner had a known-bad behaviour behind an `allocators` argument, and
one consumer in the tree was already passing the empty set at three call sites. Somebody had hit
exactly this defect, worked out the override, fixed it for their own consumer, and left the library
and its four other consumers alone.

That override is evidence at its own provenance tier: it establishes that the defect is real, that it
was reachable in practice, and — from the commit that introduced it — roughly when somebody knew. It
is also the strongest argument for repairing the library rather than the rows, because a workaround
applied per consumer is a rule with copies, and the copies will diverge.

Two habits follow:

- **On confirming a defect, grep for its disable flag, its magic constant, and its name.** An
  existing override names the defect, the consumer that noticed, and the date.
- **When you repair the library, fold the overrides back in and say so.** Leaving them is how the
  repaired behaviour gets disabled again at three call sites nobody is looking at, and the next
  measurement of the defect reads as a regression.

---

## Two witnesses that trace to the same fact are ONE witness

A confidence column counts corroborating witnesses, so the question that decides the column is not
"how many rules agreed" but "how many independent FACTS did they read". Rules phrased differently,
run by different tools, against different artifacts, can still rest on one measurement.

Measured: a class's size had two upper-bound routes that looked independent —

1. a derived class places its first own member at offset `0x64`, and the compiler puts the first
   derived member at `roundup(sizeof(base), align)`;
2. a separately-initialised 4-byte static sits at `instance + 0x64`, and two distinct static objects
   cannot overlap.

Route 2 is what an *earlier round of the same project* had already used to set that size, expressed
against an artifact column (`art_slot - descriptor == 0x64`) rather than against the disassembly.
Same offset, same object, same fact, arrived at from the other side. Only route 1 was new.

Counting both would have recorded a corroborated size resting on one witness, and — because route 2
traces back to this project's own earlier apply — would also have re-imported a number the round was
supposed to be testing independently. **Trace each witness to the concrete fact in the binary it
rests on, and dedupe on the FACT, not on the rule.** When two routes reach the same offset by
different arguments, say which byte each one read; if it is the same byte, you have one witness and a
cross-check, which is worth recording as exactly that.

---

## Before adding a route to a witness kind, grep the file that CONSUMES the artifact, not just the one that produces it

A census of *"what does this witness reach"* naturally reads the producer. The second
implementation is usually not there.

Measured: a harvest sweep gained a new route to an existing witness kind, and the ADJUDICATOR that
consumes its output had carried its own version of the same idea for several rounds — scoped to a
hand-approved literal of exactly one class, with its own reach print and its own raises. The census
had read the sweep and never opened the decider, so the collision was invisible until the decider
refused the run outright: *"already carries a row — a second row for one class is what the reader
raises on"*.

**The resolution is the reusable part, and it is not deletion.** The older route YIELDS to the
harvest and becomes a CROSS-CHECK. A hand-approved, body-level census from an earlier round is the
strongest calibration a general rule can have — it was approved against the binary by a human, it
names an address and a value, and the general rule had no access to it. Asserted rather than
deleted, it reported **exact agreement, 1 of 1**, on the same body and the same number.

Two details that make the assert real rather than decorative:

- **State the DIRECTION the two may differ in.** Both were lower bounds from the same body, and the
  harvest folded strictly more writes in, so the harvested value may be LARGER and may never be
  smaller. That is a one-sided check and it catches a regression in either route.
- **Fix the messages the older route prints.** Its reach line described the overlap as *"already
  carry a row from <the old route>"*, which stopped being true the moment a second producer existed,
  and it printed an unexplained `DISAGREE` that was simply the difference the new route exists to
  create. **A false statement in a gate's own output is a defect even when no number moves** — the
  next reader will treat it as a contradiction between two routes that are not the two named.

## A namespace move on a DEFAULT symbol stays DEFAULT — a provenance channel no SourceType tier marks

Read from Ghidra 12.1.2's `SymbolDB.setNameAndNamespace` (SoftwareModeling-src.zip): when a symbol's
source is `DEFAULT`, a namespace change clears the stored name, keeps the dynamic `FUN_` name, and
**keeps the source `DEFAULT`**. So moving an unnamed function under a class -- the way its `__thiscall`
auto-`this` gets typed -- leaves no `SourceType.AI` mark. A gate that grades AI-sourced names sees
neither the move nor an analyzer moving it back. The same path is taken by a **name withdrawal**: a
name set back to DEFAULT keeps whatever namespace it had.

Give these moves their own ledger and a both-directions gate over *DEFAULT-named functions in a
non-global namespace*. Measured on its first run in one project: 17 already present, 3 of them
withdrawn names that had kept their class namespace, the rest library namespaces whose origin
nobody had recorded. Origin: re-metal-fatigue §421.

## A NOTE THAT SAYS "CLOSED" IS A CLAIM ABOUT A DECISION RULE — open the rule before building on it

Measured on one 1999 MSVC/x86 binary: a round recorded a pointer cell's refinement (base class ->
derived class) as "closed by an export-typed chain", on the strength of ONE caller passing an exported
getter's typed return to the setter that fills the cell. The next round, asked to teach the producing
sweep that chain, first read the sweep's decision rule (a refinement needs two independent BODIES) and
every call site of the setter: the export typing held at 1 of 5 non-NULL callers; another caller merged
a typed return with an untyped handle-table load. Nothing contradicted the refinement, but "closed" was
false and had already reached a backlog item and a round record. The fix was a user decision to relax
the bar deliberately (two typings through different exports at different sites, over a supertype that
already clears the bar), not a rule bent to fit. **"Closed" names the decision rule's verdict; read
that rule, not the note.**

Related, same round: **a probe's "undecided" bounds the probe.** A register-liveness probe abstained at an
indirect jump; reading the jump table from the PE and each case's first instructions settled it in
minutes, cited per deciding instruction.

## A VERDICT ANSWERS A QUESTION — record the question beside it, or the verdict becomes a property of the thing

The previous section is about a note that was false. This one is about a note that was TRUE and still
misled, because it was read at a wider scope than it was written at.

Measured on the same binary: a round classifying struct designs for a re-flattening sweep wrote that
structs modelling their base as one opaque member are *"a third design and never a repair target"* --
correct for that sweep, which has nothing to refresh inside an opaque member. Three rounds later a
different question reached the same structs: a class whose own vtable is longer than the one its
embedded base reaches needs its own vtable struct at offset 0, and an embedded base member owns offset
0. The queued item now read as a *reversal of a decision*, and was priced as one, when the design had
simply met a question the earlier round never asked. Nothing in the earlier round was wrong; its verdict
had lost its question on the way into the notes.

- **Write the question into the sentence that carries the verdict** -- *"never a repair target FOR THE
  RE-FLATTENER"* costs four words and turns a later reversal into a routine round.
- **When a queued item says a round would "reverse" an earlier decision, first find the question the
  earlier round was answering.** If it is a different question, there is nothing to reverse, and the
  write-up should say so rather than record a reversal that never happened.
- The census still had to be run: pricing found the affected population was 26 classes and not the 2 the
  item named, of which 24 bought nothing observable. The value of the census was the split, and the
  user chose the smallest option. Origin: re-metal-fatigue §425.

## When two witnesses disagree, look for the one neither produced — and check provenance markers in both directions

A width convention from GUI registration strings ("X/Y/Z without a `.Layer` component means a 12-byte
vector") disagreed with a 16-byte serialisation record for one member, and the project correctly carried it
as an unresolved contradiction for rounds: neither channel could outvote the other from inside. A third
witness from outside both settled it in one artifact join: the virtual methods the class overrides sit at
slots whose exported mangled signatures take and return `const CLVector &`, the override block-copies four
dwords into the member, and an unrelated exported function takes the same member by that reference. The
resolution changed the RULE (a missing component decides nothing about width), measured first to un-type
nothing that rule had already claimed. The general form: a declared signature at a shared virtual slot is
an independent type witness for every override, and a contradiction between two inference channels is a
prompt to search for it, not a tie to break.

The same round wrote a false "name not from the binary" marker onto a name the binary spells, because the
post-apply check asserted only that invented names KEEP the marker, never that spelled names do not GAIN
it. A provenance assertion that runs in one direction certifies half the claim. (Origin: re-metal-fatigue
§430.)


## A calibration set the licensed repair shrinks is a countdown, not a calibration

A structural detector was calibrated on the rows whose answer the binary states independently
(demangled export prototypes the analyzer under-modelled), and the repair the detector licenses
rewrites each such row in the project's own tier -- so every repair removed a calibration row: 25,
then 20, then, worked to the end, only the rows the detector must NOT fire on. The floor moved with
the set each round and every gate stayed green, because a floor pinned to a shrinking population is
consistent by construction. The obvious replacement -- the project's own ledger of repaired rows,
each "proven" by several witnesses -- is the self-harvest trap in a new coat: the detector is one of
those witnesses, so it would grade itself on its own verdicts. What works is a population nothing
in the loop can reach: the export table's by-value returns, fixed by the linker and read from an
artifact the program cannot influence, which kept every repaired row exactly as it kept the
unrepaired ones (6 of 6 repaired, all detected). Two rules. **Ask of a calibration population "what
moves it?" -- if the answer is "the repair it licenses", the number tracks the work's progress, not
the rule's truth.** And a one-sided calibration (every row is a positive) still needs its negatives
from somewhere; here they are exactly the rows no repair ever removes, so that arm does not consume
itself either. (Origin: re-metal-fatigue §436.)
