# grilling-crew

A Claude Code skill that converts any structured task document into implemented,
tested, and verified code — using a full multi-agent pipeline.

## What it does

You give it a .md file with tasks, fixes, or features to implement.
It spins up a crew of specialized AI agents that work in parallel, implement
everything, review the result, run verification tests, and update the doc with
the final status of every task.

## The pipeline

Phase 1  →  Analyst-Agent + CodebaseExpert-Agent run in parallel
             (requirements clarity + deep codebase audit)

Phase 2  →  Orchestrator synthesizes findings, defines safe parallel batches

Phase 3  →  Slave agents implement tasks in parallel (conflict-free file ownership)

Phase 4  →  Review-Agent checks all modified files for syntax and logic errors

Phase 5  →  Live verification tests run per task

Phase 6  →  Task file updated with final status for every task

## Installation

Clone directly into your global Claude Code skills folder:

bash
# macOS / Linux
git clone https://github.com/YOUR_USERNAME/grilling-crew ~/.claude/skills/grilling-crew

# Windows
git clone https://github.com/YOUR_USERNAME/grilling-crew $env:USERPROFILE\.claude\skills\grilling-crew

Available immediately in every project on your machine. No restart needed.

## Usage

/grilling-crew [task-file] [spec-file]

**task-file** — path to your .md file with the task list (required — skill will ask if omitted)
**spec-file** — path to a PRD or spec doc for requirement validation (optional)

If you call /grilling-crew with no arguments, the skill asks you for both files
before starting.

## Recommended workflow

Pair with /grill-with-docs for the best results:

1. /grill-with-docs     →  stress-tests your plan, produces a solid task doc
2. /grilling-crew       →  crew implements everything in that doc

Grill first. Crew second. Never feed a vague task doc to the crew.

## What the crew handles automatically

Tasks marked **deferred** or **out of scope** — documented but never implemented
**Data gaps** — flagged as info entries in the output, never patched via data files
**Read-only directories** — respected as hard constraints throughout
**Shared-file conflicts** — tasks that touch the same file always go to the same agent

## Requirements

[Claude Code](https://claude.ai/code) (CLI or IDE extension)
A structured task document in .md format

## File structure

grilling-crew/
├── SKILL.md
└── references/
    ├── agent-roles.md       # Orchestrator / Analyst / CodebaseExpert / Slave definitions
    ├── batching-guide.md    # How to group tasks into conflict-free parallel batches
    └── pipeline-phases.md  # Expanded checklists for each pipeline phase

## License

MIT
