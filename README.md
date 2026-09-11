# agent-skill

Senior-level engineering standards and engineering log for AI coding agents.

This repository contains two companion artifacts designed to be loaded into an AI coding agent's context to make it write code the way a senior staff engineer would.

## Contents

| File | Purpose |
|------|---------|
| [`SKILL.md`](./SKILL.md) | The skill: 19 sections of opinionated rules covering frontend, backend, auth, AI/LLM integration, security, performance, observability, code review, incident response, and anti-patterns. |
| [`PROBLEMS.md`](./PROBLEMS.md) | The post-mortem log: a living record of non-routine problems, root causes, solutions, and durable lessons. Includes 10 illustrative seed entries and copy-paste templates. |

## What is this?

### SKILL.md

A decision filter, not a checklist. It tells the agent:

- **An operating loop** (Anchor → Scope → Design → Implement → Verify → Close) that grounds every change.
- **A conflict-resolution hierarchy** (user requirements → repo conventions → security/correctness → maintainability → performance).
- **Section-specific rules** for full-stack + AI work, each with concrete examples rather than generic advice.
- **A failure-modes catalog** mapping symptoms to fixes for recurring bugs.
- **Hard anti-patterns** that end discussions (`catch (e) {}`, float-for-money, string-concatenated SQL, prompt-injection defense by obscurity, etc.).

### PROBLEMS.md

Institutional memory. Every non-routine problem gets an entry with:

- **Context** — what triggered the problem
- **Problem** — concrete symptoms and measured numbers
- **Root cause** — why it actually happened (not just the symptom)
- **Solution** — what was changed
- **Lesson** — the rule to follow next time
- **Severity**, **Status**, **Tags**, **Related** — for filtering and cross-linking

Pattern detection is built in: after ~30 entries, recurring bugs become visible and can be converted into lint rules, shared tests, or SKILL.md updates.

## How to use

### For AI agents

Load `SKILL.md` as a system-prompt or skill definition. Reference `PROBLEMS.md` when reasoning about past failures or designing preventive measures.

### For humans

Read `SKILL.md` to understand the standards the agent will apply. Maintain `PROBLEMS.md` as the team's post-mortem log — every non-trivial bug fix or incident should add an entry.

## Contributing

1. Update `SKILL.md` when a new rule emerges from repeated `PROBLEMS.md` patterns.
2. Add `PROBLEMS.md` entries for non-routine bugs, regressions, performance surprises, security events, or LLM-specific issues.
3. Keep the `PROBLEMS.md` index table synchronized with new entries.
4. Replace illustrative seed entries with real project history once the repo is active.

## Review cadence

- **Weekly:** skim new `PROBLEMS.md` entries for patterns.
- **Monthly:** review top 3 most-severe entries; verify follow-ups.
- **Quarterly:** update `SKILL.md`, ADRs, and runbooks based on lessons learned.

## Author

Maintained by [bikash-20](https://github.com/bikash-20).
