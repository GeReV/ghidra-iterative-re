# Platform and era reference

Lookup detail for non-PC and non-modern targets. `SKILL.md` carries the triggers — *set
register context or whole regions disassemble as garbage*, *an address is not an identity in
an overlaid or bank-switched program*, *mark hardware registers volatile* — and this file is
what you consult once one fires.

## Architectures and where they show up

**Endianness first, before anything else.** The same processor family runs big-endian on
some systems and little-endian on others, so the architecture alone does not determine the
Ghidra language variant. Choosing wrong (`PowerPC:BE:32:default` vs the LE variant) silently
garbles the whole disassembly, and nothing you do later recovers from it — you re-import.
Big-endian: GameCube, Wii, Xbox 360, PS3 (PowerPC); Saturn (SH-2); Genesis, Amiga, classic
Mac (68k); N64 in its common ROM byte order. Little-endian: PS1, PS2, PSP (MIPS); DS, GBA
(ARM); all x86. Verify against known constants or a recognisable string rather than trusting
the platform's reputation.

| Architecture | Systems | Notes |
|---|---|---|
| x86 16-bit | DOS, early Windows | Segmented addressing; MZ/NE/LE formats; overlays common |
| x86 32-bit | Windows, Xbox (original) | PE; `__thiscall`/`__stdcall`/`__fastcall` all appear |
| MIPS | PS1, PS2 (+VUs), N64, PSP | `$gp`-relative data addressing; delay slots |
| PowerPC | GameCube, Wii, Xbox 360, PS3 | Big-endian; TOC/`r2`-relative on some ABIs |
| SH-2 / SH-4 | Saturn, Dreamcast | PC-relative constant pools |
| 68000 | Genesis, Amiga, classic Mac, arcade | Big-endian |
| ARM / Thumb | GBA, DS, 3DS, mobile | **Mode switching is the main hazard** |
| 6502 / Z80 / SPC700 | NES, SNES audio, 8-bit era | Bank switching; tiny address spaces |

Console executable and ROM formats frequently need a community loader extension. Failing
that, import raw with a correct base address and hand-built memory map — but record that
you did, because every address in the program then depends on that choice.

## Register context — the difference between code and garbage

Some architectures encode per-region processor state that the disassembler must be told:

- **ARM/Thumb mode.** The `TMode` context register selects instruction set. Wrong value →
  the whole region decodes as nonsense. Set it over the address range rather than fighting
  individual instructions.
- **MIPS `$gp`.** Data accesses are `$gp`-relative; without the correct value, global
  references resolve nowhere. Set the register value over the function or region.
- **PowerPC `r2`/TOC** on ABIs that use it, similarly.

Set these with a register-value-over-range operation before disassembling, not after.
Re-disassembling a region after fixing context is normal and expected.

## Overlays and bank switching — address is not identity

**PS1 and DOS games routinely load different code to the same address at different times.
Cartridge systems bank-switch ROM into a window.** In both cases one address holds several
unrelated things over the program's life.

Consequences that break the default methodology:

- Model each mapping as a Ghidra **overlay memory block**, so each gets its own address
  space. Without that, analysing one overlay silently corrupts the other's disassembly and
  types.
- **Every address-keyed artifact becomes ambiguous.** This methodology leans on joining
  evidence by address; in an overlaid program the key must be *(overlay space, address)*.
  Audit any evidence CSV for this before trusting a join.
- A function count invariant computed across overlays counts things that never coexist at
  runtime. Still useful as a change detector; not meaningful as a population.

## Memory-mapped hardware registers

Writes to fixed addresses that are not memory at all: VDP/GPU registers, DMA controllers,
audio chips, controller ports, timers.

- **Label them from platform documentation.** A single recognised register write identifies
  the surrounding code's purpose instantly — this is often the fastest way into an unknown
  console binary.
- **Mark them `VOLATILE`** (`MutabilitySettingsDefinition`). Otherwise the decompiler is
  entitled to fold repeated reads of the same address into one, which silently deletes the
  poll loop you were trying to read.

## Era-specific arithmetic

- **Fixed-point.** On pre-FPU and console targets, an integer multiply followed by a right
  shift is fixed-point math, not a bug and not an integer computation. The shift amount is
  the fractional bit count — a shift of 16 is Q16.16, 12 is Q20.12, 8 is Q24.8. Ghidra
  shows no float type anywhere; division often appears as a shift, and a "weird" constant
  like `0x10000` is `1.0`. Recovering the Q format converts a wall of shifts into readable
  geometry.
- **x87 (1990s x86).** Floating point runs on an 8-deep register *stack*, so arguments and
  results move by stack discipline (`FLD`/`FSTP`) rather than named registers. Decompiler
  output is awkward; reading argument order requires tracking pushes. Presence of `FLD`/
  `FMUL` on a struct field is strong evidence that field is a float — a useful type witness
  even when the decompiler renders it badly.
- **SSE-era (2000s onward).** The same math looks entirely different — packed `XMM`
  operations, often four lanes at once, so one instruction may touch four struct fields.
  Do not infer field widths from instruction widths without checking for packing.

## Calling conventions worth recognising

- **x86**: `__cdecl` (caller cleans, args pushed **right-to-left** — so the *first* argument
  is pushed **last**, i.e. nearest the `CALL` and easiest to recover by a backward walk;
  it is the *last* argument that sits furthest away and is hard to reach),
  `__stdcall` (callee cleans), `__thiscall` (`this` in `ECX`, MSVC), `__fastcall`
  (`ECX`/`EDX` then stack).
  *An earlier revision of this file stated that backwards.* And a caveat that matters more
  than the direction: at `/O2` MSVC frequently writes arguments with `MOV [ESP+N], …`
  instead of `PUSH`, so a walk that counts pushes under-reads the argument list entirely
  rather than reading it in the wrong order.
- **Large struct returns**: MSVC passes a hidden return-storage pointer, so a by-value
  return of a big struct appears as `T * f(this, T *__return_storage_ptr__)`. This changes
  a function's signature the moment you apply the struct type — a known source of
  harvester non-idempotency.
- **MIPS o32/n32/n64**, **PowerPC**: register-passed arguments with an argument-area
  convention; check the compiler spec Ghidra selected (`-cspec` headless) rather than
  assuming.
- Custom conventions exist and Ghidra supports declaring them; the course material covers
  Program Specification Extensions and callfixups for cases where a function violates the
  ABI (hand-written assembly, stack adjustment stubs).

## Copy protection and packing

Packed, encrypted, or self-modifying code appears in commercial games of every era. Signs:
a tiny entry point, high-entropy sections, sections that are written before being executed,
imports resolved at runtime. Ghidra's **entropy sidebar** gives a quick read on which
regions are compressed or encrypted. For decode routines, the **p-code emulator**
(`EmulatorHelper`) runs them without the game and yields the plaintext, which is usually
faster than reasoning about the algorithm.
