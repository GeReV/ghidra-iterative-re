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
