# ghidra-iterative-re

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) for
reverse-engineering binaries in Ghidra via MCP or PyGhidra.

Ghidra is not a disassembly viewer you read from. It is an inference engine with a
persistent, versioned database: knowledge you apply changes what the decompiler shows you
next, which is where the next round of knowledge comes from. Refusing to write to it
throws away the tool's main mechanism.

But that loop has a failure mode batch analysis does not — **you can corroborate your own
guesses.** Apply an inference, re-read, and the confirming read looks independent when it
is not. This skill is mostly about making the loop safe against that, and against
Ghidra's silent collateral damage.

## Status: work in progress

This is a living document, not a finished reference. It is still being written *by* the
project that uses it — every round of that work either adds a rule or corrects one, and
that has not converged.

Concretely, what that means for you:

- **Rules get withdrawn.** Several claims here replaced earlier ones that measurement
  refuted. Assume the same will happen to some of what is currently written.
- **Coverage is uneven.** It is deepest where the source project went deep — MSVC/x86
  PE, C++ single inheritance, class layout recovery. Other platforms and toolchains get
  a reference file's worth of triggers, not battle-tested procedure.
- **It reorganizes.** `SKILL.md` has been restructured more than once and sections may
  move or split; don't deep-link into it.
- Claims marked *(doc)* are from Ghidra's documentation and **have not been executed**.

Corrections are more welcome than additions, particularly if you have a measurement that
contradicts something here. See [Provenance](#provenance-and-how-to-read-the-numbers) for
why the specific numbers in it are not thresholds.

## What's in it

`SKILL.md` is the loop and the rules that bind every round. Everything that is a
*catalogue* lives in `references/`, keyed by what you are about to do.

| File | Contents |
|---|---|
| `SKILL.md` | The methodology: the round loop, the `SourceType` trust model, invariant bracketing, ceremony-vs-blast-radius, the free-names checklist, and a routing table to everything below. |
| `references/harvesting-traps.md` | 62 measured traps in reading evidence back out: self-harvest, blank censuses, skip counters, reach pricing, the byte-pattern engine. |
| `references/assertions.md` | 53 on assertion discipline under iteration: checks that cannot fire, cannot pass, or go inert as the program moves underneath them. |
| `references/trust-and-circularity.md` | The deep end of the trust model: laundering paths, the TYPE axis (types have no `SourceType` at all), and why a tier cannot tell two sources apart that share it. |
| `references/applying-changes.md` | What is worth applying, emitting a C header with compiled offset assertions, and the rest of the mutation-safety bracket. |
| `references/game-recon.md` | Free names, separating library code from game code, data shapes, modern engines, platform breadth, runtime observation. |
| `references/oracles-and-abi.md` | The external oracle — the only check that is not the program grading itself — and target-vs-host pointer width. |
| `references/api.md` | Ghidra API surface verified against a real 12.1.2 install, the MCP boundary, read-only witnesses, cascade triggers, and the PyGhidra scripting rules. |
| `references/cpp-abi.md` | MSVC/Itanium name mangling, vtable and vbtable symbols, modelling classes via `ClassUtils`, dispatch recovery and devirtualization. |
| `references/platforms-eras.md` | Non-PC and pre-modern targets: register context, bank switching, overlays, volatile hardware registers. |
| `references/sources.md` | Citations, the external-oracle reading list, and which claims are bound to a particular Ghidra version. |

Only `SKILL.md` is loaded when the skill is invoked; the references are read on demand
when one of the triggers in `SKILL.md` fires. It used to be 3,029 lines and carry all of
the above; the split happened when the rules that bind *every* round stopped being
findable among the ones that bind a particular kind of round.

Some of the load-bearing ideas:

- **The `SourceType` trust model** — harvest evidence *excluding* `SourceType.AI`, or a
  second run eats its own output and the result merely looks richer.
- **Invariant bracketing** — diff a whole-program invariant across every mutation and
  `raise` on any unaccounted change, in either direction. Ghidra destroys unrelated
  functions silently; nothing warns you.
- **Assertion discipline** — every assertion must be demonstrated *failing* before it
  counts, and so must every evidence source be demonstrated *producing a row*. A check
  that cannot fire is indistinguishable from a check that passed.
- **Cascade is the stage that gets skipped** — it is the only stage that leaves no
  artifact whose absence you can notice, so it needs a forcing function.
- **Match the ceremony to the blast radius** — a read-only sweep and a 1400-function
  apply do not deserve the same process.

## Install

Clone it, then link it into your skills directory:

```sh
git clone https://github.com/GeReV/ghidra-iterative-re.git
ln -s "$PWD/ghidra-iterative-re" ~/.claude/skills/ghidra-iterative-re
```

A symlink means `git pull` updates the installed skill with no second copy to go stale.
If your setup can't follow symlinked skill directories, `cp -r` works too — but then
re-copy after every update.

Invoke it with the `Skill` tool as `ghidra-iterative-re`, or `/ghidra-iterative-re`.

It is not Claude-Code-specific in substance: `SKILL.md` is plain Markdown and reads fine
as a methodology document for a human or any other agent harness.

## Provenance, and how to read the numbers

This was not written from first principles. It was derived from — and continuously
corrected by — a long-running reverse-engineering project against a 1999 MSVC/x86 game
binary, over roughly 5600 recovered functions and several phases of work. Almost every rule in it
exists because something went wrong first, and most of them say so.

That has one consequence worth stating up front, which `SKILL.md` repeats:

> **Quoted numbers from one binary are examples for calibration, not properties of
> yours.** Claims marked *(doc)* come from Ghidra's own documentation but have not been
> executed; unmarked claims were either measured live or are project history.

So `656 rep-string instructions` or `12 of 14 anchors` are there to give you a sense of
scale and of what a real measurement looks like. They are not thresholds to check your
own binary against.

## License

MIT — see [LICENSE](LICENSE).
