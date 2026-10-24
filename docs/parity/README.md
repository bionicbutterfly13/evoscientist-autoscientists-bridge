# EvoScientist paper vs code parity

Audit of the EvoScientist technical report ([arXiv:2603.08127](https://arxiv.org/abs/2603.08127)) against the released code at [v0.3.8, commit e2eef14](https://github.com/EvoScientist/EvoScientist/tree/e2eef14), done on 2026-10-24.

Three read-only audits, one per part of the paper:

| Report | Paper sections | Scope |
|---|---|---|
| [researcher.md](researcher.md) | 3.3, 4.5 | Researcher agent, ideation memory, idea tree search, Elo tournament, proposal |
| [engineer.md](engineer.md) | 3.4, 4.5, RQ2 | Engineer agent, experimentation memory, four-stage experiment tree search |
| [evolution.md](evolution.md) | 3.1, 3.2, 3.5, 5.4 | Evolution manager, IDE/IVE/ESE, memory design, ablations |

Status labels: **Implemented** (coded logic), **Partial**, **Prompt-only** (an LLM is instructed to do it; nothing checks it happened), **Divergent** (something different is built), **Missing**.

## Summary

The paper's mechanisms are not in the EvoScientist repository as code. They exist as LLM instructions in the separate [EvoSkills](https://github.com/EvoScientist/EvoSkills) skill pack (`research-ideation`, `experiment-pipeline`, `evo-memory`), which users install through `/evoskills` or onboarding. The repository itself ships a general agent harness with a different memory design (profile files, typed observations, a self-linking observation graph, AutoSkills).

Main gaps, ranked by effect on reproducing the paper's results:

1. **No coded researcher, engineer, or evolution manager agent.** The paper's three agents are replaced by one orchestrator plus six task sub-agents. Idea direction evolution, idea validation evolution, and experiment strategy evolution are prompt text only.
2. **No ablation switches.** The −IDE, −IVE, −all and −ESE arms (Table 3, Fig. 2) cannot be toggled. The available memory toggles disable a different system.
3. **No execution success-rate harness.** Nothing computes per-stage execution success, so the RQ2 numbers cannot be regenerated.
4. **No tree search in code.** Per-stage attempt budgets (20/12/12/18), best-node selection, and stage-to-stage hand-off are unenforced prose. Current upstream `research-ideation` has also dropped the idea tree: it refines 3 tracks and runs a 3-match round-robin instead of an Elo tournament over up to 21 candidates.
5. **No embedding retrieval.** No `mxbai-embed-large`, no cosine top-k (k_I=2, k_E=1). The repository uses lexical ranking; the skills use LLM judgement.
6. **Ideation and experimentation memory are workspace-local markdown files.** In `--mode=run` each run gets a fresh folder, so cross-task evolution can fail to carry over.
7. **Trigger and scope differ.** Repository memory workers fire after every turn and sub-agent run. The skill runs strategy evolution only after all four stages succeed, so failed trajectories never reach experimentation memory.
8. **Human selects the idea to extend.** The paper extends the top Elo idea automatically.
9. **One model for all roles.** The paper used Gemini-2.5-Pro for ideation and Claude-4.5-Haiku for code. The default is `claude-sonnet-5-5` for everything, with no per-sub-agent model setting.
10. **No tests** for ideation, tournament, retrieval, or evolution.

Spot-checked against source before publishing: `EvoScientist/prompts.py:59-66` (skill sequence), `EvoScientist/config/settings.py:247` (default model), no `cosine`/`mxbai` in the package, and `evo-memory/SKILL.md:125` (ESE trigger on success only).

## What this means for the bridge

The bridge cannot rely on EvoScientist to enforce experiment budgets, record dead ends, or switch evolution on and off. Those have to live in the bridge's own tested Python, which is what the AutoScientists queue, claim and result protocol already provides. See the integration plan (to be added under `.planning/`).

## Open questions

- Which EvoSkills commit produced the paper's results. The local install is commit `6d92ea5` (v2.1.0); upstream is v2.1.2.
- Whether "initial implementation success" in RQ2 means "runs" or "matches reported metrics".
- Ranking criterion for best-node selection (paper Eq. 8).
