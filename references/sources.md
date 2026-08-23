# Sources, and version-sensitivity

*Reference for the `ghidra-iterative-re` skill. Only `SKILL.md` is loaded when the
skill is invoked; this file is read on demand when its trigger fires. New lessons of
this kind belong here, not in `SKILL.md`.*

**Read this when:** you want the citation behind a claim here, or need to know which
claims are bound to a particular Ghidra version.

## Sources

Ghidra 12.1.2 local install: `docs/GhidraClass/Advanced/improvingDisassemblyAndDecompilation.pdf`
(vftable recipe, data mutability, signature commit semantics, Decompiler Parameter ID
warning, non-returning functions, switch/flow overrides), `docs/WhatsNew.md` (12.1
bitfield recovery, Objective-C call overriding, Jython-as-extension, JDK 21),
`docs/ghidra_stubs/pypredef/` (`AutoAnalysisManager`, `RefType`, `SourceType`,
`EmulatorHelper`, BSim packages), `docs/GhidraAPI_javadoc.zip`.

- [SourceType javadoc](https://ghidra.re/ghidra_docs/api/ghidra/program/model/symbol/SourceType.html) · [GhidraScript.AnalysisMode](http://ghidra.re/ghidra_docs/api/ghidra/app/script/GhidraScript.AnalysisMode.html)
- [Ghidra #650](https://github.com/NationalSecurityAgency/ghidra/issues/650), [#516](https://github.com/NationalSecurityAgency/ghidra/issues/516) — no automatic devirtualization
- [Override Call Reference](https://grant-h.github.io/docs/ghidra/decompiler/classOverride.html) · [decompiler override.cc](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/Decompiler/src/decompile/cpp/override.cc)
- [DemanglerCmd source](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/Base/src/main/java/ghidra/app/cmd/label/DemanglerCmd.java)
- [GhidrAssistMCP source](https://github.com/symgraph/GhidrAssistMCP) — audited for `SourceType` usage; [issue #66](https://github.com/symgraph/GhidrAssistMCP/issues/66) proposes `SourceType.AI`
- [LLM Agent-Assisted RE with Quantitative Readability Metrics](https://arxiv.org/html/2606.06838) — metric gaming, coverage, context rot
- [Agentic RE: Building Custom AI Skills with Coding Agents, Recon 2026](https://cfp.recon.cx/recon-2026/talk/SHYHKM/)
- [und3rf10w ghidra-scripting agent skill](https://skillsmp.com/creators/und3rf10w/ai-ghidra-tools/plugins-ghidra-skills-ghidra-scripting)
- [Beginners Guide to Reverse Engineering (Retro Games)](https://www.retroreversing.com/tutorials/introduction) · [Reverse Engineering a GBA Game](https://www.starcubelabs.com/reverse-engineering-gba/) · [Reverse engineering save game files](https://mytechnologyblog532.wordpress.com/2016/11/13/reverse-engineering-save-game-files/) · [Unity IL2CPP save file RE](https://blog.painite.ch/en/blog/unity-save-file-reverse-engineering/)
- This project: `notes/tooling-capabilities.md`, `notes/assertion-discipline.md`,
  `notes/phase2-report.md` §5

**External oracles and second opinions** (added after an expert review of this document,
2026-08-15; the review verified most API claims against a 12.1.2 install, and the web
sources below are cited at the tier the review gave them):

- [isledecomp/reccmp](https://github.com/isledecomp/reccmp) + [isle](https://github.com/isledecomp/isle) — matching decompilation for **32-bit x86 MSVC**, with `vtable`/`datacmp`/`stackcmp`. The closest existing methodology to this skill.
- [encounter/objdiff](https://github.com/encounter/objdiff) — x86-capable interactive object differ. (`m2c`, `decomp-permuter`, `asm-differ` are **not** x86.)
- [CMU SEI Pharos / OOAnalyzer](https://github.com/cmu-sei/pharos) · [CCS'18 paper](https://edmcman.github.io/papers/ccs18.pdf) · [CERT Kaiju](https://github.com/cmu-sei/kaiju) — C++ class recovery for MSVC x86 **without RTTI**.
- [NCC Group — EarlyRemoval in the Conservatory with the Wrench](https://www.nccgroup.com/research/earlyremoval-in-the-conservatory-with-the-wrench-exploring-ghidra-s-decompiler-internals-to-make-automatic-p-code-analysis-scripts/) — why a p-code harvester must declare its simplification style.
- [Votipka et al., *An Observational Investigation of Reverse Engineers' Processes*, USENIX Security 2020](https://www.usenix.org/system/files/sec20-votipka-observational.pdf) — expert REs work hypothesis-first and lean on **control-flow beacons** over name/string beacons.
- [Ghidra VT workflow help](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/VersionTracking/src/main/help/help/topics/VersionTrackingPlugin/VT_Workflow.html) — NSA's own certainty ladder (Symbol Name → Exact Data → Exact Function Bytes → Exact Instructions → Exact Mnemonics → Duplicate Function → reference correlators, which consume already-*accepted* matches). Two warnings worth obeying: *scores are not comparable between correlators*, and *do not modify either program while version tracking*.
- [Skochinsky, *Reversing MSVC Part II: Classes, Methods and RTTI*](https://www.openrce.org/articles/full_view/23) — the canonical MSVC object-layout reference.
- [BinDiff](https://github.com/google/bindiff) + [BinExport](https://github.com/google/binexport) — structural, call-graph-propagating matching that survives changes VT's exact correlators reject. (The tagged BinExport Ghidra extension targets 11.0.3; expect to rebuild for 12.x.)
- [REBench](https://arxiv.org/abs/2604.27319) · [REFORGE](https://arxiv.org/abs/2607.07738) · [OSPREY](https://yonghwi-kwon.github.io/data/osprey_sp21.pdf) — measured ceilings for LLM-assisted RE and Ghidra's own type-recovery baselines.
- [ReVa](https://github.com/cyberkaida/reverse-engineering-assistant) — Ghidra-12-native agent tooling shipped as skills; direct prior art.
- [Il2CppInspectorRedux](https://github.com/LukeFZ/Il2CppInspectorRedux) · [Il2CppDumper](https://github.com/Perfare/Il2CppDumper) · [gdsdecomp](https://github.com/GDRETools/gdsdecomp) — modern-engine dumpers.

**Version-sensitivity.** API claims here were checked against **12.1.2** and several are
version-bound: source-map APIs are 11.3+, `SourceType.AI` is recent, PyGhidra became the
default in 12.0 (with 3.0 deprecations), `SymbolicPropogator`'s recording default changed in
**11.4.1** (`docs/ChangeHistory.md:678` — this document said 12.0 for several revisions), and
`AddVfunctionCallRefScript` is 12.1. **Re-verify against your install before
relying on any of them** — the same discipline as stamping evidence rows with a program
version, applied to the tool.
