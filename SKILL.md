---
name: ghidra-iterative-re
description: Use when doing reverse engineering in Ghidra through MCP or PyGhidra scripts - establishes the apply-cascade-verify loop, the SourceType trust model that prevents an agent corroborating its own guesses, and the invariant bracketing that catches Ghidra's silent collateral damage.
---

# Iterative reverse engineering in Ghidra

Ghidra is not a disassembly viewer you read from. It is an inference engine with a
persistent, versioned database. Knowledge you apply to the program changes what the
decompiler shows you next, which is where the next round of knowledge comes from.
Refusing to write to it — reading disassembly and xrefs only — throws away the tool's
main mechanism and is the more expensive choice.

But the loop has a specific failure mode that batch analysis does not: **you can
corroborate your own guesses.** Apply an inference, re-read the program, and the
confirming read looks independent when it is not. This skill exists to make the loop
safe, not to make you cautious about using it.

## The loop

Work in **rounds**. Each round has a fixed shape, and you never skip a step:

```
checkpoint  →  apply (highest-certainty tier only)  →  cascade (scoped)
            →  verify invariants  →  harvest  →  adjudicate  →  next round
```

Stop when a round produces nothing new (two consecutive dry rounds), not when you run
out of ideas.

**Apply in certainty order, never in convenience order.** Round N applies only what is
more certain than what round N+1 will apply. Ground truth from the binary first
(export tables, mangled names, RTTI, debug data), then mechanical derivations from it,
then inferences. A round that applies a guess before an available certainty has
corrupted every round after it.

## The trust model — this is the load-bearing part

Ghidra tracks provenance natively via `SourceType`, and Ghidra 12 added a level
specifically for agents. Priority order, highest first:

| SourceType | Means | May it be used as evidence? |
|---|---|---|
| `USER_DEFINED` | A human decided this | Yes — but record that a human is the source |
| `IMPORTED` | Came from the binary: export table, demangler, debug info | **Yes — this is ground truth** |
| `ANALYSIS` | Ghidra's own analyzers inferred it | Yes, with the analyzer named |
| `AI` | *"content produced through AI assistance"* — **your** inferences | **NEVER** |
| `DEFAULT` | `FUN_`/`DAT_` placeholders | No — absence of information |

**Tag every mutation you make as `SourceType.AI`.** Then the anti-circularity rule
stops being a discipline you must remember and becomes a query filter:

> **Harvesters must exclude `SourceType.AI` symbols from evidence.**

This is the whole defense. A round's own applications structurally cannot become
evidence for the claim that motivated them, because the harvester cannot see them.
Verify the filter fires by tagging one known symbol `AI` and confirming it drops out
of the harvest.

Corollary: **prefer scripts over MCP mutation tools when provenance matters.** A
script sets `SourceType` explicitly; an MCP rename/retype tool usually defaults to
`USER_DEFINED`, which silently launders your inference into ground truth. Before using
any MCP mutating tool, verify what `SourceType` it writes — do not assume.

## Invariant bracketing — because Ghidra damages things silently

**Every mutation is bracketed by an invariant diff, measured before and after, and it
`raise`s on change. Never prints — raises.**

This is not paranoia; it is a recorded incident. `ApplyFunctionDataTypesCmd` called
with a broad address set against a large type archive **destroyed two unrelated
functions** — disassembly and function definition both gone, every byte reverted to
undefined — with **no error, no exception, and no log line.** `applyTo()` returned
cleanly. It was caught only by diffing the total function count afterward
(2640 → 2638).

So:

- Pick a cheap whole-program invariant (function count is the usual one) and diff it
  across every mutating operation.
- Count the same way every time and write down which way. Ghidra offers more than one
  function count and they differ (in this project: `getFunctions(True)` → 2640
  excludes externals; `getFunctionCount()` → 2774 includes them).
- **A clean return and no exception is not evidence nothing was damaged.**
- **Scope every address set as narrowly as the operation allows.** The incident above
  did not reproduce when the same archive was applied to a narrow set. Broad address
  set plus large archive is the dangerous combination.
- Gate every mutating script behind an explicit `apply` argument so a bare run is a
  dry run.

## Checkpoint, and know your rollback works

Rollback makes mutation cheap — but only if it works, and "the snapshot file was
written" is not evidence that it does.

- Checkpoint before every round: end the script's transaction, `save()`, snapshot,
  then the version-control step. Ordering matters — a snapshot taken before a pending
  save silently captures the *previous* state while claiming to be current.
- A script *can* check the program in. The sequence is `end(True)` →
  `currentProgram.save(comment, monitor)` → `domainFile.checkin(handler, monitor)`.
  If checkin reports unsaved changes, that message is literal: save first.
- **Exercise a restore before you need one.** An archive that passes an integrity
  check is not a proven rollback path. Restore once, early, deliberately.

## What actually cascades — and what does not

Be precise here, because the temptation is to overclaim propagation:

**Cascades, genuinely:**

- **Applying function signatures.** Parameter and return types propagate to call sites
  in the decompiler. This is the strongest real cascade. Mangled names from an export
  table encode full signatures; `DemanglerCmd(addr, mangled).applyTo(program, monitor)`
  applies both name and signature.
- **Defining functions at previously-undefined code.** New functions give the
  analyzers new material, extend pointer-table runs that were truncated by undefined
  targets, and add call-graph edges. Often the highest-leverage single mutation.
- **Applying struct layouts.** Field accesses become named. Note this can *change*
  function signatures as a side effect — a large struct returned by value switches to
  the hidden return-storage-pointer convention, so `T Func(this)` becomes
  `T * Func(this, T *__return_storage_ptr__)`.

**Does NOT cascade — do not plan around it:**

- **Virtual call devirtualization.** Typing a vtable does *not* make the decompiler
  resolve `(**(code **)(*(int *)this + 0xa0))()` to a concrete target. Ghidra cannot
  prove the vtable pointer is constant after assignment. You get better *rendering*
  and a named slot; you do not get a resolved call edge. Naming the target function is
  a separate, real win — it just arrives by a different mechanism.

### Triggering the cascade, scoped

Auto-analysis is `ENABLED` by default during scripts ("Auto-Analysis responding to
changes"), so mutations may trigger analysis *while your loop runs* — meaning you can
read half-analyzed state. For determinism, mutate first and analyze deliberately:

- `analyzeChanges(program)` — "Starts auto-analysis if not started and waits for
  pending analysis to complete." **This is the sanctioned cascade trigger.**
- `analyzeAll(program)` — full re-analysis of the entire program. Reckless on a
  curated program; it can undo curation. Avoid.
- For surgical scope, `AutoAnalysisManager.getAnalysisManager(program)` exposes
  `codeDefined(addr|set)`, `functionDefined(addr|set)`,
  `functionSignatureChanged(addr|set)`, `dataDefined(set)`,
  `createFunction(target, findFunctionStart)`, `getAnalyzer(name)`,
  `cancelQueuedTasks()`. Tell it exactly what changed, then `analyzeChanges`. Note
  this class is not in the public javadoc; it is reachable but unsupported API.
- `GhidraScript.AnalysisMode`: `ENABLED` (default, analysis responds live),
  `SUSPENDED` (queued, analyzed after the script), `DISABLED` (change events ignored).

## Assertion discipline under iteration

The standing rule — *an assertion you have not seen fail is not evidence* — still
holds, but iteration changes which checks are cheap and which are dangerous.

- **Prefer checks that already fail.** If ground truth from the binary contradicts the
  current state, those disagreements are free demonstrations that the check fires. Use
  them instead of constructing poison.
- **Poison where `expected` and `actual` might share a source.** That is the failure
  shape: two reads of the same variable in the same loop, compared against each other,
  producing a comparison that cannot disagree with itself.
- **Never verify against state you produced this round.** The `SourceType.AI` filter
  enforces this for harvesters; apply the same rule by hand to any ad-hoc check.
- **Never verify against an artifact you know to be wrong.** If a dataset has been
  measured incorrect where it is checkable, it is disqualified as a reference — even
  for the parts you have not checked.
- Test verification logic against poisoned *copies of files*, never by mutating the
  live program.

## Provenance across rounds

**Harvesters are not idempotent across an apply.** After you apply knowledge, a
harvester re-run against the same program can yield *different* evidence than it did
before — not because it is broken, but because the program legitimately changed. A
signature-matching harvester that looked for `T Func(...)` stops matching once the
apply rewrites that signature to `T * Func(..., T *__return_storage_ptr__)`.

Consequences, and they are structural, not edge cases:

- **Stamp every evidence row with the program version it was derived at.** Evidence
  from round N is not reproducible at round N+1 and must not be silently regenerated.
- **Treat earlier evidence as append-only.** Regenerating an evidence file after an
  apply can silently drop whole witness categories while looking like a clean re-run.
- When a harvester must run post-apply, teach it to accept the post-apply shape as
  well as the pre-apply one — do not just re-run it and trust the output.

## Operating rules (MCP)

- **Editing a script in your repo does nothing.** Re-install it into Ghidra's user
  script directory before every run (`action="create"`, `overwrite=true`). This
  includes library modules: Ghidra's script directory sits earlier on `sys.path` than
  an appended repo path, so a stale installed copy of a library shadows your edit
  silently. You will debug logic that is not running.
- **`action="run"` is asynchronous.** It returns a task id. Nothing is verified until
  you have read `get_task_status`. Never report a result you have not read.
- A `create` response may misreport the provider (`<unsupported>`, `Jython`). Cosmetic
  — the `run` result reports the real one.
- Prefer scripts over per-item MCP tool calls for bulk work: one script run beats
  hundreds of round-trips, and only the script can set `SourceType`.

## Scripting rules (PyGhidra)

- Scripts are **CPython 3 via PyGhidra**, not Jython. `except X as e`, f-strings, real
  `import`. A `# @runtime Jython` tag fails before the first line executes.
- Wrap database modification in a transaction; `GhidraScript` opens one for you, and
  `end(True)` closes it (required before save/checkin).
- Dispose decompilers and close files explicitly.
- Pass long operations the `monitor` so they can be cancelled.
- **Discover API from the local stubs, not from memory or the web.** A Ghidra install
  ships `docs/ghidra_stubs/pypredef/` (hundreds of files) and
  `docs/GhidraAPI_javadoc.zip`. These match your exact version, which web
  documentation may not. Search them with Python rather than shell text tools if your
  environment's `grep` is proxied or unreliable.
- Ghidra writes to the host filesystem with plain `open()`. On Windows-hosted Ghidra
  driven from WSL, script source must use Windows paths (`C:\...`) while you read the
  same files from `/mnt/c/...`.

## Failure modes to design against

Drawn from community research on agent-driven RE as well as this project's own
history:

- **Metric gaming.** Given a single score to optimize, agents optimize the score.
  In one study agents discovered that *minimizing changes* maximized a binary-
  similarity metric, and reverted variables to `rax`/`rcx` — the measure became the
  target and stopped measuring. Use multi-dimensional criteria, and never let
  "verification passed" be the objective.
- **Coverage theatre.** The same study found only 10–15% of functions received genuine
  analysis while the run reported completion. State coverage as a number, always.
- **Context rot across a long run**, producing inconsistent naming between functions
  analyzed early and late. Short, scoped executions beat one monolithic session.
- **Distinguish measured-zero from structural-zero.** "I looked and found nothing" and
  "there was nothing to look at" are different findings. Conflating them wastes the
  next round or buries a real absence. Emit counters that separate them.
- **A reproducible error is not evidence for your theory about its cause.** It is
  evidence the same trigger fires every time. Read the error text literally first.
- **Volume is not the metric.** A confidently wrong result is worse than an absent
  one, because later rounds treat it as evidence.

## Sources

- Ghidra 12.1.2 API: `SourceType`, `GhidraScript`, `GhidraScript.AnalysisMode`,
  `AutoAnalysisManager` (via local `pypredef` stubs), `DemanglerCmd`
- [Ghidra SourceType javadoc](https://ghidra.re/ghidra_docs/api/ghidra/program/model/symbol/SourceType.html)
- [GhidraScript.AnalysisMode javadoc](http://ghidra.re/ghidra_docs/api/ghidra/app/script/GhidraScript.AnalysisMode.html)
- [Ghidra issue #650 — decompile virtual function calls as C++ style calls](https://github.com/NationalSecurityAgency/ghidra/issues/650)
- [Ghidra issue #516 — seeing vtable function calls in the decompiler](https://github.com/NationalSecurityAgency/ghidra/issues/516)
- [DemanglerCmd source](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/Base/src/main/java/ghidra/app/cmd/label/DemanglerCmd.java)
- [LLM Agent-Assisted Reverse Engineering with Quantitative Readability Metrics](https://arxiv.org/html/2606.06838) — metric gaming, coverage, context rot, micro-prompt findings
- [Agentic Reverse Engineering: Building Custom AI Skills with Coding Agents (Recon 2026)](https://cfp.recon.cx/recon-2026/talk/SHYHKM/)
- [und3rf10w ghidra-scripting agent skill](https://skillsmp.com/creators/und3rf10w/ai-ghidra-tools/plugins-ghidra-skills-ghidra-scripting) — PyGhidra vs Jython patterns, FlatProgramAPI, disposal/transaction hygiene
- This project: `notes/tooling-capabilities.md` (the `ApplyFunctionDataTypesCmd`
  incident, checkpointing, stale-import shadowing), `notes/assertion-discipline.md`
  (the six unfireable assertions), `notes/phase2-report.md` §5 (harvester
  non-idempotency across an apply)
