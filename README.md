# OpenEval

A Claude Code skill that helps you build an evaluation harness for an AI agent.

## What's in here

- **`agent-eval-harness/`** — the skill. `SKILL.md` is the entry point; `references/` holds depth on each design decision (first principles, simulation environment, scoring, result vs trace, dataset construction).

## What it does

When invoked, the skill:

1. Reads your agent's codebase to learn what it does.
2. Asks only what the code can't answer (production failure modes, what you actually want to improve).
3. Walks you through four design decisions: simulation environment, scoring approach, result vs trace, dataset construction.
4. Writes a harness tailored to your agent — file names and types speak your domain, not generic placeholders.

It teaches as it builds. The goal is that after one pass you understand the harness well enough to extend it yourself.

## Install

Symlink into your global Claude Code skills directory:

```bash
git clone https://github.com/softpudding/OpenEval.git
ln -s "$PWD/OpenEval/agent-eval-harness" ~/.claude/skills/agent-eval-harness
```

Then in any Claude Code session, ask something like *"help me build an eval harness for my agent"* or invoke `/agent-eval-harness`.

## Principles

Two, from `references/principles.md`:

1. The harness measures what the agent is actually for.
2. The harness localizes failures, not just produces an aggregate number.

A harness that violates either will mislead the team building the agent.
