# Parity audit: EvoScientist paper §3.3 Researcher Agent vs code

Paper: arXiv 2603.08127, §3.1 (Stage 1 definition), §3.2, §3.3 (Eq. 1–5), §4.5, App. A/D, Fig. 2.
Code: [EvoScientist/EvoScientist](https://github.com/EvoScientist/EvoScientist/tree/e2eef14) v0.3.8 (read-only; nothing edited).

## Provenance note (read first)

The code repo contains no researcher-agent logic. Ideation lives entirely in an external, separately installed skill pack (EvoScientist/EvoSkills), which the main agent is only told to use:

- `EvoScientist/prompts.py:61` names skill `research-ideation` ("Explore the field, rank candidate ideas, produce a research proposal").
- `EvoScientist/config/onboard/steps.py:1103-1106` recommends installing `EvoScientist/EvoSkills@skills`; repo `EvoScientist/skills/` ships only `find-skills` and `skill-creator`.

Two versions of that skill were inspected (both are prompt text, no scripts for tree search or Elo):

- **Installed** (local): `~/.evoscientist/skills/research-ideation/` v2.1.0 (files dated Aug 14), plus `~/.evoscientist/skills/evo-memory/` v1.0.0. Cited below as `[inst] path:line`.
- **Upstream HEAD**: EvoScientist/EvoSkills, last research-ideation commit `cb61d4904b` (2026-09-30), v2.1.2, fetched read-only via `gh api` to scratchpad/upstream/. Cited as `[up]`. Upstream **deleted** `references/tree-search-protocol.md` (404) and replaced the Swiss tournament over up to 21 candidates with a 3-champion round-robin.

"Prompt-only" below means: the mechanism exists only as LLM instructions in these external SKILL.md files, with no coded logic in the repo.

## 1. Claim table

| # | Paper claim | Paper ref | Status | Code evidence | Note |
|---|---|---|---|---|---|
| 1 | A distinct Researcher Agent (RA) generates ideas and proposals | §3.2, §3.3, Fig. 1 | Divergent | `subagents/research.yaml:1-3` (research-agent = web-search assistant, tools `[tavily_search, think_tool]`); `prompts.py:61` | No RA agent exists; the single main agent follows the `research-ideation` skill, and the sub-agent named `research-agent` is a literature/web lookup helper, not the RA. |
| 2 | Persistent ideation memory M_I | §3.3 ¶1, Fig. 2 | Prompt-only | [inst] `evo-memory/SKILL.md:31-46`; `research-ideation/SKILL.md:56`; repo `EvoScientist.py:890-898` routes only `/memories/`; searched repo for `ideation`, `ideation-memory`, `idea_memory`: only `prompts.py:61`, test display strings `tests/test_stream_utils.py:138,208` | M_I is a markdown file `/memory/ideation-memory.md` the LLM reads and writes; it resolves to the workspace/run sandbox (`EvoScientist.py:867,874`), not the global `/memories/` store, so cross-task persistence depends on reusing the same workspace. |
| 3 | M_I records feasible AND unsuccessful directions | §3.2, §3.3 ¶1 | Prompt-only | [inst] `evo-memory/assets/ideation-memory-template.md:6,28`; `evo-memory/SKILL.md:37-44` | Template has "Feasible Directions" and "Unsuccessful Directions" sections with fundamental vs implementation failure classification; no schema enforcement in code. |
| 4 | Retrieve_I uses embedding-based cosine similarity | §3.3 Eq. 1 | Divergent | [inst] `evo-memory/SKILL.md:159` ("perform this semantic comparison by reading each entry's Summary… and Retrieval Tags, then judging relevance"); [up] same text line 159; repo `memory/search.py:153-191` (lexical IDF/regex over observations, unrelated to M_I); searched: `cosine`, `embed`, `embedding`, `mxbai`, `vector`, none in repo | Retrieval is LLM judgment over the whole file; the skill itself admits the paper used embeddings and substitutes manual relevance judging. |
| 5 | top-k_I most similar items, k_I = 2 | Eq. 1, §4.5 | Prompt-only | [inst] `research-ideation/SKILL.md:57`; `evo-memory/SKILL.md:164` | k_I=2 is stated as an instruction to the LLM; nothing enforces or measures it. |
| 6 | Embedding model mxbai-embed-large via Ollama | §4.5 | Missing | searched: `mxbai`, `embed`, `ollama` (repo has Ollama only as a chat-model provider, `llm/models.py:662`, `llm/ollama_discovery.py`) | No embedding model is configured or called anywhere. |
| 7 | Tree-structured propose–review–refine search | §3.3 Eq. 2, Fig. 2 | Prompt-only (installed) / Divergent (upstream) | [inst] `research-ideation/references/tree-search-protocol.md:52-67` (per-node Propose→Review→Refine, 3-level technique/domain/formulation tree); [inst] `SKILL.md:127-148` (3 tracks × 3 personas × N=3 iterations); [up] tree-search-protocol.md removed, SKILL.md "Track champion = the refinement picked in the final iteration" | Installed skill describes two inconsistent procedures (a 3-level tree in the reference file, linear 3-track persona refinement in SKILL.md); upstream keeps only the linear 3-track refinement, which is not a tree. |
| 8 | Each node stores (i) idea draft and (ii) review feedback rev_i; outputs {(I_i, rev_i)} | §3.3 Eq. 2 | Partial (prompt-only) | [inst] `research-ideation/assets/idea-candidate-template.md` sections `Tree Position, Description, Novelty Argument, Feasibility Assessment, Tournament Record, evo-memory Check` (no review/feedback field); `SKILL.md:205-210` "Refinement Summary" only for top-3 | Review happens in-context, but no per-node review record is persisted or passed as a structured pair. |
| 9 | N_I = 21 maximum candidate ideas | Eq. 2, §4.5 | Prompt-only (installed) / Divergent (upstream) | [inst] `tree-search-protocol.md:69-74` ("Up to N_I=21 Leaf Candidates"); [up] `elo-ranking-guide.md` "Why only the champions? Entering every refined version… (up to 27)… was tested" ; upstream tournament input = 3 champions | Upstream deliberately abandoned the 21-candidate pool, so current code cannot reproduce the paper's candidate count. |
| 10 | 3 parallel workers for idea tree search | §4.5 | Prompt-only | [inst] `SKILL.md:129` ("Run 3 parallel refinement tracks"); repo `prompts.py:393` (generic advice to fan out `task()` calls, mentions "ELO-style tournaments"); searched: `max_workers`, `worker` in ideation context, none | No worker pool or concurrency setting for ideation; parallelism is at the LLM's discretion. |
| 11 | Search grounded in retrieved literature L (Semantic Scholar API) | Eq. 2, §4.5 | Prompt-only (external) | [inst] `SKILL.md:70-93` (MUST use paper-navigator, 30–50 papers, challenge-insight tree); [inst] `paper-navigator/scripts/utils.py:38` (`https://api.semanticscholar.org/graph/v1`); repo: searched `semantic scholar`, `semanticscholar`, none; repo research-agent uses `tavily_search` (`subagents/research.yaml:3`) | Semantic Scholar access exists only in the external paper-navigator skill scripts; the repo's own search path is Tavily web search, which the ideation skill forbids for paper discovery. |
| 12 | Grounded in retrieved memories K_I (feasible seeds, failed pruned) | Eq. 2 | Prompt-only | [inst] `SKILL.md:58-59`; `tree-search-protocol.md:89-91` | Seeding and pruning by M_I are instructions only. |
| 13 | Elo-based tournament with pairwise comparisons | §3.3 Eq. 3 | Prompt-only | [inst] `SKILL.md:155-172`; `references/elo-ranking-guide.md:9-27` (E_A formula, update rule); searched repo: `elo`, `tournament`, `pairwise`; only hit `prompts.py:393` (generic) | Elo arithmetic is done by the LLM in text; no code computes ratings. |
| 14 | Elo settings (initial rating, K-factor, pairing, rounds) | not stated in paper | N/A (paper silent) | [inst] `SKILL.md:170` (start 1500, K=32), `elo-ranking-guide.md:92-119` (Swiss, 4–5 rounds); [up] round-robin of 3 champions, 3 matches total | Paper gives no Elo hyperparameters; installed and upstream skills disagree on pairing and pool size. |
| 15 | Ranking criteria: novelty, feasibility, relevance, clarity | §3.3 before Eq. 3 | Prompt-only | [inst] `SKILL.md:159-166`; `elo-ranking-guide.md:40-90` (equal 25% weights) | Matches paper's four dimensions. |
| 16 | Retain top-3 ideas I_top for direction summarization | Eq. 4 | Prompt-only | [inst] `SKILL.md:174-236` (present top-3, save `/direction-summary.md`, trigger IDE) | Matches in intent; with upstream's 3-champion pool, "top-3" is every candidate, so the filter is vacuous. |
| 17 | Extend the top-1 idea into proposal P automatically | Eq. 5 | Divergent | [inst] `SKILL.md:215-219` ("Which idea would you like to develop… (1/2/3, or combine elements)"), `SKILL.md:238-240` ("After the user selects an idea") | Skill inserts a human choice (and allows combining ideas) instead of auto-extending the Elo winner. |
| 18 | Proposal P contains background, related work, method, experimental plan, expected results | §3.1, §3.3 after Eq. 5 | Prompt-only | [inst] `research-ideation/references/proposal-extension.md:5,27,42,98,131` (Sections 1–5 exactly these) plus `:156` Risks; `assets/proposal-template.md:33` domain-adaptive universal sections | Matches paper; skill adds Risks and a field-specific template step. |
| 19 | Idea I = brief method description + experimental plan | §3.1, §3.3 | Prompt-only | [inst] `SKILL.md:115-125` ("Core Idea" + "Validation Plan") | Matches. |
| 20 | Ideation LLM is Gemini-2.5-Pro | §4.5 | Divergent (configurable) | `config/settings.py:246-247` default `anthropic` / `claude-sonnet-5-5`; `llm/registry.py:149` lists `gemini-2.5-pro` | Paper model is selectable but not the default, and there is no per-stage model routing for ideation. |
| 21 | Tests covering ideation / Elo / retrieval | (reproducibility) | Missing | searched `tests/` for `elo`, `tournament`, `ideation`, `idea`: only skill-manager fixtures (`tests/test_skills_manager.py:795`) and display strings (`tests/test_stream_utils.py:138,208`) | No tests exercise any researcher-agent mechanism. |

## Hyperparameters stated by the paper (§4.5)

| Param | Paper | Code / skill |
|---|---|---|
| k_I | 2 | Prompt text only ([inst] `SKILL.md:57`) |
| N_I | 21 | Prompt text in installed `tree-search-protocol.md:69`; removed upstream (3 champions) |
| Ideation parallel workers | 3 | "3 parallel tracks" prompt text; no setting |
| Embedding | mxbai-embed-large via Ollama | Absent |
| Ideation model | Gemini-2.5-Pro | Default claude-sonnet-5-5; Gemini selectable |
| Literature source | Semantic Scholar API | External paper-navigator skill only |
| Elo init / K | not stated | Skill: 1500 / 32 |

## 2. Missing or divergent, ranked

1. **No coded researcher pipeline at all (claims 1, 7, 13).** Every RA mechanism is LLM instructions in an external, versioned skill pack the user must install separately, so runs are not deterministic or reproducible and nothing checks that the steps actually happened.
2. **Candidate pool and tournament changed upstream (claims 9, 14, 16).** Paper ranks up to N_I=21 tree candidates with Elo; current upstream ranks 3 track champions in a 3-match round-robin and says the larger tournament was tested and dropped, so the paper's idea-quality results (Table 1) were produced by a different procedure than the current skill.
3. **Not a tree search (claims 7, 8).** Upstream replaced the propose–review–refine tree with linear 3-persona refinement per track, and no version stores per-node review feedback rev_i.
4. **Embedding retrieval absent (claims 4, 6).** Retrieve_I is LLM judgment over the whole M_I file rather than mxbai cosine top-k; behaviour changes as M_I grows and k_I=2 is not enforced.
5. **Human picks the idea to extend (claim 17).** Eq. 5 auto-extends the Elo top-1; the skill asks the user to choose or combine, which changes which proposal reaches the engineer.
6. **Models and literature source differ by default (claims 11, 20).** Default model is Claude Sonnet, not Gemini-2.5-Pro; the repo's own search tool is Tavily, and Semantic Scholar is reachable only through the external paper-navigator skill.
7. **M_I scope (claim 2).** `/memory/ideation-memory.md` lives in the workspace (or run folder) sandbox, not the global `/memories/` store, so "cross-task" evolution only holds when tasks share a workspace.
8. **No tests (claim 21).** Nothing in `tests/` guards any of the above.

## 3. Searched but uncertain

- **Which skill version produced the paper results.** Paper (Mar 2026) predates both inspected skill versions; the installed v2.1.0 (with N_I=21 tree protocol) looks closest, but EvoSkills git history before v2.1.0 was not traced.
- **Installed v2.1.0 is internally inconsistent.** `SKILL.md` Step 3–5 runs 3 tracks and ranks "track champions", while `tree-search-protocol.md` and `elo-ranking-guide.md` assume 15–21 leaves with Swiss pairing; which path an LLM follows in practice was not tested.
- **`/memory/` resolution in `--mode=run`.** Sandbox root is `work_dir` (`EvoScientist.py:867`), which can be a per-run folder (`paths.py:170-180`); whether M_I therefore resets per run was inferred from code, not run.
- **paper-navigator "Workflow 9: Ideation Support".** Referenced by [inst] `research-ideation/SKILL.md:72`; grep of the installed paper-navigator `SKILL.md` found no "Workflow 9", so the hand-off target may not exist in that version.
- **Local modification of installed skills.** Installed files differ from upstream; I attributed differences to version (v2.1.0 vs v2.1.2), not local edits, without checking an install manifest.
