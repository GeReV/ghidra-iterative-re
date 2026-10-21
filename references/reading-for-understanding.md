# Reading for understanding — bottom-up, then top-down, and keep what each read learned

**Open this when** you are about to send readers (human or model) through function bodies, or when a project has
read a lot and can still say little about what the program does.

> **Status: a method under trial.** Proposed and first piloted in the Metal Fatigue project, round §537
> (2026-10-21). The numbers below are that project's; the pilot's measurements are added as they come in. Treat
> the procedure as a hypothesis with a stated test, not as established practice.

## The failure it addresses

A project can read a binary several times over and still be unable to say what most of it does. Measured in the
project that prompted this: the decompiled C of the whole program was ~1.5–1.9M tokens, waves of readers had
spent many times that, and **the behaviour of 54 of 5,655 functions (1%) was written down**. Each read did
produce understanding, but its output was squeezed into one decision row (a name, a field type, a prototype)
and everything else the reader understood was discarded with its context. Meanwhile the project's tooling grew
to 4.6× the size of the program it studied, because the metrics it drove were mechanical database counts.

Three habits produce this, and each is reasonable on its own:

- **One column per read.** A naming wave, then a typing wave, then a globals wave, each re-reading the same
  bodies and re-building the same context.
- **Every reader starts cold.** The trust model rightly forbids counting your own applied names as evidence; if
  that is implemented as "never show a reader anything we concluded", nothing ever accumulates.
- **Understanding is not a measured output.** If the denominators are names and types, rounds are designed to
  move names and types.

## The method

1. **Scope to what is not understood, plus what it depends on.** Targets are the functions with an open item
   (no name, untyped or untrustworthy C, a global only they decide). Dependencies are everything the targets
   call, transitively, that is not itself a target. Library / runtime / import callees are a boundary: their name
   is their summary. In the first project this cut the reading to ~1,580 of ~3,340 game functions.
2. **Order bottom-up.** Collapse recursion into units (strongly connected components); a unit's layer is one
   more than the highest layer it calls; read layer 0 (the leaves) first. A reader then sees one body plus its
   callees' SUMMARIES, never their bodies, so understanding propagates upward instead of being re-derived.
   **Include virtual calls** where the receiver type is known (the decompiler's `obj->vftable->Slot` spelling
   plus the vtable layout), binding to the receiver class's OWN slot body: the slot's contract is what a caller
   needs. Binding to every override instead tripled the dependency set in the first project (435 → 1,272)
   through base-class calls that fan out to hundreds of overrides. Count what does not resolve; do not guess it.
3. **Every read leaves a durable record**: a 1–3 sentence summary in domain terms, parameters and return,
   fields and globals read and written, what each call is for, side effects, instruction-address evidence, a
   confidence, open questions — and a **`basis`** list of the callee summaries it relied on, so a summary later
   found wrong can be traced to everything built on it. Decision rows (names, types) come out of the same read.
   Store the records in the **binary's address order**, not reading order, so they are searchable and diffable.
4. **Roll up**: cluster / class summaries from function records, subsystem summaries from those, one program
   overview last. A context window holds a few hundred functions; whole-program understanding has to be built
   in layers or it is not built at all.
5. **Then go top-down, and repeat.** Bottom-up gives each function its mechanism; only its callers give it its
   purpose (`insert into list` is "queue the next build order" once you have read who calls it). A top-down pass
   re-reads summaries — not bodies — from the subsystem summaries downward, correcting purposes and resolving
   the edges the bottom-up order could not see (field receivers, untyped dispatch). Alternate until a pass
   changes nothing that matters. The `basis` column is what makes a correction cheap: it names exactly which
   records to revisit.

## Trust, kept

Summaries are your own work. A later reader may use them **as context**, and records that it did; they are
never independent **evidence** for a decision, which still needs instruction-level witnesses from the body
(`trust-and-circularity.md`). This keeps the protection against self-corroboration without the amnesia.

## Ceremony, proportionate

A summary changes nothing in the program, so it needs none of the mutation apparatus (canary, replay tests,
invariant bracket). It needs a **sampled adversarial verifier**: re-read a fraction of each batch's bodies
against the records and grade them correct / incomplete / wrong, with a wrong-rate that sends a batch back.
Decision rows keep the full pipeline, applied once at the end of the wave.

## Measure it

Make "functions in scope with a verified summary" a denominator beside the mechanical ones, and report cluster
and subsystem coverage. What is not measured is not designed for.
