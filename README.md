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

## What's in it

| File | Contents |
|---|---|
| `SKILL.md` | The methodology. Part 1 is the loop (any binary), Part 2 is what to point it at in a game, Part 3 is the tool surface. |
| `references/api.md` | Ghidra API surface verified against a real 12.1.2 install, plus the offline doc paths to discover more. |
| `references/cpp-abi.md` | MSVC/Itanium name mangling, vtable and vbtable symbols, modelling classes via `ClassUtils`, devirtualization. |
| `references/platforms-eras.md` | Non-PC and pre-modern targets: register context, bank switching, overlays, volatile hardware registers. |

Only `SKILL.md` is loaded when the skill is invoked; the references are read on demand
when one of the triggers in `SKILL.md` fires.

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
git clone https://github.com/<you>/ghidra-iterative-re.git
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
