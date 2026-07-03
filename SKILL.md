---
name: grilling-crew
description: Use this skill when the user invokes /grilling-crew, when the user wants to implement all tasks or fixes from a .md document using a full multi-agent pipeline, when the user says "run the crew", "start the grilling crew", "implement everything in this file", "deploy the crew on this", or "apply all fixes from the doc". Works best after /grill-with-docs or /to-prd has produced a structured task document. Accepts an optional path to a task file and an optional path to a spec/PRD document.
version: 1.0.0
---

# Grilling Crew — Multi-Agent Implementation Pipeline

A 7-phase orchestrated pipeline that converts a structured task document (fixes, features, refactors, migrations) into implemented, tested, and verified code changes — using a crew of specialized sub-agents working in parallel where safe.

## Invocation

```
/grilling-crew [task-file] [spec-file]
```

- **task-file** *(required)*: Path to the `.md` file containing the task list.
- **spec-file** *(optional)*: Path to a PRD, ADR, or spec document (PDF or `.md`) that the Analyst-Agent will use to validate requirements.

---

## Step 0 — Argument Collection (ALWAYS run this before anything else)

Before reading any file or launching any agent, check what arguments were provided.

### If `task-file` is missing or was not provided:

Use `AskUserQuestion` to ask:

```
Question: "Which file contains the tasks or fixes to implement?"
Header: "Task file"
Options:
  - "Let me type the path" (user will provide via Other)
  - Suggest any .md files you can spot in the project root or a docs/ folder
    (run a quick Glob for *.md in the working directory first)
```

Do not proceed until the user provides a valid path. Verify the file exists (Read it) before continuing.

### If `spec-file` is missing or was not provided:

Use `AskUserQuestion` to ask:

```
Question: "Is there a spec or PRD document the Analyst should use to validate requirements?"
Header: "Spec / PRD"
Options:
  - "No spec — work from the task file only"
  - "Yes — let me provide the path" (user will provide via Other)
  - Suggest any PDF or .md files in docs/ that look like specs
```

If the user selects "No spec", set spec-file = null and proceed. The Analyst-Agent will work from the task file alone.

### Once both are resolved:

Confirm to the user in one line:

> "Task file: `[path]` | Spec: `[path or 'none']` — starting the crew."

Then proceed to **Your Role: Orchestrator** below.

---

## Your Role: Orchestrator

You are the **Orchestrator**. You do not write code directly. You read reports, synthesize them, make decisions, assign work to sub-agents, and verify results. Think like a tech lead who coordinates a team.

**Before starting Phase 1:** Read the task file in full. Identify:
- What *type* of tasks are these? (bug fixes / feature additions / refactors / migrations / mixed)
- How many tasks, grouped by priority if priorities exist
- Which files are mentioned across multiple tasks (shared-file conflicts)
- Any tasks explicitly marked as deferred or out-of-scope

If the task file content is ambiguous after reading, stop and ask the user the specific question — do not guess.

---

## Phase 1 — Parallel Intelligence Gathering

Launch **Analyst-Agent** and **CodebaseExpert-Agent** in a single message (parallel, same Agent tool call batch). Do not wait for one before launching the other.

### Analyst-Agent prompt template

> You are an expert requirements analyst.
>
> Task file: `[TASK_FILE_PATH]`
> Spec file (if provided): `[SPEC_FILE_PATH]` — read pages 1-20 first, continue if needed.
>
> Read both documents in full. Produce a structured report:
> 1. For each task (in priority order): what is the requirement, what is currently broken or missing, and the exact business impact.
> 2. Cross-task dependencies — which tasks must be completed before others.
> 3. Tasks that are explicitly **deferred or out of scope** — flag these clearly, do not implement them.
> 4. Tasks that are **data gaps** — they cannot be fixed by code changes alone.
> 5. Hard constraints mentioned in the task file (e.g. read-only directories, API limits, architectural rules).
> 6. Your recommendation on safe parallelism: which tasks are file-independent and can run concurrently.
>
> Be specific. Quote the task file where relevant.

### CodebaseExpert-Agent prompt template

> You are a senior developer doing a full codebase audit before implementation.
>
> Task file: `[TASK_FILE_PATH]`
>
> Read the task file completely to understand every task. Then read these files (use Read tool, not just grep — you need to see the full context):
> [LIST ALL FILES MENTIONED IN THE TASK FILE]
>
> Also explore related files that the task file's changes will interact with.
>
> For each task, produce:
> 1. The **exact current code** at the lines/functions mentioned (quote it — do not paraphrase).
> 2. **Discrepancies** between what the task file says the code looks like and what it actually looks like.
> 3. **Variable names and function signatures** that slave agents will need (so they use correct names).
> 4. **Cross-file dependencies**: if task A changes a function, which other files call that function?
> 5. **Risk assessment**: what existing behaviour could break with each change?
> 6. The **exact replacement code** each task requires, adjusted for what the code actually looks like.
>
> Be exhaustive. Slave agents will rely on your report — any mistake here propagates.

---

## Phase 2 — Synthesis and Approach

After both agents return, synthesize their findings:

1. **Identify conflicts**: which tasks touch the same file? Those must go to the *same* slave agent and be applied sequentially within that agent.
2. **Identify safe parallelism**: tasks in different files with no shared dependencies can go to different slaves running in parallel.
3. **Flag deferred items**: record them — you will update the task file at the end but will NOT implement them.
4. **Flag data gaps**: note them as evidence-only — document in code (e.g. as a flag in the output) but do not attempt to modify data files.
5. **Define batches**: group tasks by file ownership. See [references/batching-guide.md](references/batching-guide.md) for the batching strategy.

Present a brief implementation plan (2-3 sentences per batch) before spawning slaves — this is your commit to the approach.

---

## Phase 3 — Slave Agent Dispatch

Spawn slaves in a **single message with multiple Agent tool calls** (parallel launch). For each slave:

### What every slave prompt must contain

1. **Role**: "You are an expert developer implementing a specific set of tasks."
2. **Files to modify**: the exact file paths, nothing else.
3. **The actual current code** (from CodebaseExpert-Agent's report, not from the task file — the task file may be wrong).
4. **The exact replacement code** (from CodebaseExpert-Agent's adjusted version).
5. **Sequence constraint**: if multiple tasks touch the same file, list them in order — apply them sequentially.
6. **Verification step**: after each change, read the modified block back and confirm the edit landed correctly.
7. **Hard constraints** inherited from the task file (read-only directories, deferred items, etc.)
8. **Return format**: list of `(file, what changed, any issues found)`.

### Slave batching pattern

Group tasks so each slave owns a non-overlapping set of files. If two tasks both modify `f1/nodes.py`, they go to the same slave — never split them.

See [references/agent-roles.md](references/agent-roles.md) for detailed role definitions and [references/batching-guide.md](references/batching-guide.md) for the full batching strategy.

---

## Phase 4 — Post-Implementation Review

After all slaves complete, spawn a **Review-Agent** (a new CodebaseExpert instance):

> You are a senior developer doing a post-implementation code review.
>
> The following files were modified by implementation agents:
> [LIST ALL MODIFIED FILES]
>
> Read every modified file in full. For each change:
> 1. Is the Python/JS/[language] syntax correct?
> 2. Do variable names and function signatures match the surrounding context exactly?
> 3. Is the logic consistent with what the task file intended?
> 4. Does the change introduce any obvious regressions in other callers?
> 5. Are there any naming collisions, scope issues, or import errors?
>
> Report: PASS (no issues) or a list of specific problems with file:line references.
> If problems exist, provide the exact corrected code.

If the Review-Agent finds issues, spawn targeted fix-slaves for each issue. Do not re-spawn the full slave crew — only the specific files that have problems.

---

## Phase 5 — Testing and Verification

Run the verification steps defined in the task file. If the task file provides `curl` commands, test scripts, or example invocations — run them now.

General approach:
1. Start the server/process if needed (use the `run` skill or the startup command from the task file).
2. Run each test command from the task file's verification sections.
3. For tasks where no test command is specified, run a smoke test: invoke the affected code path and confirm it returns a valid result (not an error/exception).
4. Record: PASS / PARTIAL / FAIL per task.

If a test fails, diagnose first (use the `diagnosing-bugs` skill vocabulary: build a feedback loop, form 3 hypotheses). Spawn a targeted fix-slave. Re-test.

---

## Phase 6 — Final Report and Task File Update

After all verifications, update the original task file with final status. For each task, update its status indicator:

| Result | Status to write |
|---|---|
| Implemented and verified | `✅ IMPLEMENTED & VERIFIED` |
| Implemented, not tested (no test defined) | `✅ IMPLEMENTED` |
| Partially implemented | `⚠️ PARTIAL — [reason]` |
| Skipped (deferred by task file) | `🔲 DEFERRED — out of scope` |
| Data gap — cannot fix via code | `📊 DATA GAP — [explanation]` |
| Failed | `❌ FAILED — [reason]` |

Add a **## Implementation Run Summary** section at the top of the task file with:
- Date of implementation run
- Total tasks: X implemented, Y deferred, Z data gaps
- Files modified (list)
- Any remaining work or known issues

---

## Constraints and Guardrails

These apply in every run — never override them without explicit user instruction:

- **Read-only directories**: If the task file declares any directory as read-only or immutable, never write to it. Surface data gaps as `flags[]` or log entries instead.
- **Deferred items**: Never implement a task marked "deferred", "out of scope", or "future sprint". Update its status but leave the code untouched.
- **Data gaps**: If a fix requires data that doesn't exist in a read-only source, document it as an `info`-severity flag in the output — do not seed or modify data files.
- **Destructive operations**: Never drop tables, delete files, or remove database records without explicit user confirmation.
- **Force pushes / git operations**: Never push to remote, force-push, or amend published commits without user approval.
- **Scope creep**: Slaves implement only what is assigned. They must not refactor, clean up, or add features beyond the specific task.

---

## Pairing with `/grill-with-docs`

The ideal workflow is:

```
1. /grill-with-docs  →  stress-tests your plan, produces structured task doc
2. /grilling-crew [task-file] [spec-file]  →  implements everything in the task doc
```

After `/grill-with-docs` finishes, the resulting ADR or task document is the natural input to `/grilling-crew`. Pass it as the `task-file` argument.

---

## Quick Reference

See the supporting reference files for detailed guidance:

- [references/agent-roles.md](references/agent-roles.md) — full role definitions for each agent type
- [references/batching-guide.md](references/batching-guide.md) — how to group tasks into slave batches safely
- [references/pipeline-phases.md](references/pipeline-phases.md) — expanded per-phase checklists
