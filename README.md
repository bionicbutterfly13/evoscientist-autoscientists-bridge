# evoscientist-autoscientists-bridge

Runs the AutoScientists self-organizing experiment loop (analyst, GPU and monitor agents coordinating through ClawInstitute) inside EvoScientist, as an installable skill and expert.

Status: planning. The sections below describe the intended design. Tutorial and examples will land here as the code is built.

## Why this exists

### The problem

[EvoScientist](https://github.com/EvoScientist/EvoScientist) is organized around one main agent that delegates to six sub-agents (planner, research, code, debug, data analysis, writing) and moves through intake, plan, execute, evaluate and write. That shape suits literature review, analysis and paper drafting. It is a weaker fit for work where the answer comes from running many experiments against a single metric, such as tuning a training recipe or a benchmark model over hours or days.

[AutoScientists](https://github.com/mims-harvard/AutoScientists) was built for that second kind of work. Agents form teams around hypotheses, critique each other's proposals before compute is spent, and share results and dead ends so the search stays parallel. It ships as a runbook for Claude Code: the orchestrator is a Claude Code session, every agent is a Claude Code subagent, and only Claude models run it.

### What keeping them separate costs

Running AutoScientists means leaving behind what EvoScientist provides: model choice across providers, memory that persists across sessions, scheduled runs, chat channels for checking in from a phone, and rubric grading. Each AutoScientists run starts in a fresh directory, so lessons from one run do not feed the next project.

Running an optimization task in EvoScientist means the main agent tries ideas in its own sequence. Its memory records observations, but there is no shared experiment queue, no proposal review before a run, and no team-level list of dead ends.

Porting by hand is not a copy-paste job. The runbook launches agents through Claude Code's Task tool, while EvoScientist launches them as deepagents async sub-agents. Every launch step, wait, and result harvest in the runbook has to be rewritten.

### What this repository does

It packages the AutoScientists loop as an EvoScientist skill and expert:

- The orchestrator steps (discussion, team formation, analyst and GPU cycles, champion promotion, stagnation checks) become tested Python functions that launch EvoScientist sub-agents instead of Claude Code subagents.
- Coordination still runs through a local ClawInstitute server, so the queue, claim and team protocol stay as the AutoScientists authors designed them.
- Run outcomes, including dead ends, are written to EvoScientist memory so later sessions can use them.
- EvoScientist's `/goal` and `/rubric` (v0.3.7+) can serve as the loop's exit condition.

## Dependencies

To be finalized during planning. Expected: Python 3.11+, EvoScientist 0.3.8+, Node.js 22+ for the ClawInstitute server, and the AutoScientists task definitions.

## Credits

AutoScientists: Gao, Fang and Zitnik, [arXiv:2605.28655](https://arxiv.org/abs/2605.28655). EvoScientist: [arXiv:2603.08127](https://arxiv.org/abs/2603.08127). This repository is an independent integration and is not affiliated with either project.
