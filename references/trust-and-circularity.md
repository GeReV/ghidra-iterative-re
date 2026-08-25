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
