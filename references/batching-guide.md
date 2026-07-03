# Batching Guide — How to Group Tasks for Slave Agents

The goal: maximize parallel execution while preventing two slaves from editing the same file simultaneously.

---

## The Core Rule

**One file → one slave.** If two tasks both touch `path/to/file.py`, they must go to the same slave agent, applied in priority order within that agent.

**Different files → can be different slaves** (run in parallel).

---

## Step-by-Step Batching Process

### 1. Build a file ownership map

For every task in the task file, extract the files it modifies. Example:

```
Task A  →  signal_fabric/fetcher.py
Task B  →  signal_fabric/fetcher.py, f1/aggregator.py
Task C  →  f1/report_builder.py, f1/scorer.py
Task D  →  f1/session_store.py, f1/parser.py, f1/nodes.py
Task E  →  f1/resolver.py, f1/parser.py
Task F  →  f1/nodes.py, signal_fabric/fetcher.py
```

### 2. Find shared-file conflicts

From the example above:
- `signal_fabric/fetcher.py` is touched by A, B, F → all three must go to the same slave
- `f1/parser.py` is touched by D, E → D and E must go to the same slave
- `f1/nodes.py` is touched by D, F → D and F must go to the same slave → F is already with A/B → merge: A, B, D, E, F all go to Slave-1

### 3. Form conflict-free batches

Once you find all the conflict chains, form the batches:

```
Slave-1: A, B, F, D, E  (all tasks that touch fetcher.py, nodes.py, or parser.py — merged)
Slave-2: C              (report_builder.py, scorer.py — no conflicts with Slave-1)
```

Slave-1 and Slave-2 can run in parallel. Within Slave-1, tasks are applied in priority order.

### 4. Apply priority ordering within each slave

Within a slave's task list, order by the task file's priority (P0 before P1 before P2, etc.). If two tasks in the same slave have a dependency (task B needs task A's variable to exist), order accordingly.

---

## Common Batch Patterns

These patterns emerge frequently in real projects. Use them as starting templates, then adjust for actual conflicts.

### Pattern: Backend service with layered architecture

```
Slave-A: Data layer changes  (fetchers, DB queries, parsers)
Slave-B: Business logic      (aggregators, reconcilers, calculators)
Slave-C: API / presentation  (report builders, serializers, routes)
Slave-D: Config / schema     (JSON configs, YAML, migration files)
```

If a task bridges layers (e.g. adds a field in the data layer AND the report layer), assign it to the slave that handles the lower layer, and note the upper-layer change as a second task for the presentation slave (with explicit dependency: "apply after Slave-A is done").

### Pattern: Conversation / session pipeline

```
Slave-Session: session_store.py, parser.py, nodes.py, graph.py
(These files form a pipeline — changes usually ripple across all of them together)
```

### Pattern: Routing / resolver fixes

```
Slave-Routing: resolver.py, router.py, dispatcher.py
(Routing changes are self-contained but touch a cluster of files)
```

### Pattern: Config alignment

```
Slave-Config: *.json, *.yaml, *.toml
(Config-only changes are always safe to run in parallel with code changes)
```

---

## Sequencing Between Slaves

Most slaves can run fully in parallel. Sequencing is only needed when:

1. **Slave A adds a new function that Slave B calls.** Slave B must wait for Slave A to finish and confirm the function exists with the correct signature.
2. **A shared config file** (e.g. `sources_config.json`) is modified by multiple slaves. Serialize these.
3. **Slave A fixes a bug that Slave B's task description depends on.** Slave B should be told the corrected state, but if they overlap in files, merge into one slave.

When sequencing is needed, launch the first batch, wait for completion, then launch the second batch.

---

## Slave Prompt Size Guide

Keep each slave's task list manageable:
- **Ideal**: 3-5 tasks per slave, all in 2-4 files
- **Maximum**: 8 tasks or 6 files — beyond this, spawn a second slave with a subset
- **Minimum**: No 1-task slaves unless the task is complex (e.g. adding a new helper function across many files)

If a single task is very large (e.g. "add OTT digital bridge to linear channels — requires new helper function + routing block + 5 call sites"), that task alone can be its own slave.

---

## What to Tell Each Slave About Sequencing

In the slave prompt, state explicitly:

```
SEQUENCE CONSTRAINT: Apply these tasks in this exact order within your session:
1. [Task A] — this creates the function that Task C depends on
2. [Task B] — independent, can be done in any order relative to A
3. [Task C] — depends on Task A being applied first (calls the new function)

After applying each task, read back the modified section to verify the edit landed correctly before moving to the next task.
```

---

## Anti-Patterns to Avoid

- **Splitting a file across two slaves**: never. One file = one slave.
- **Giving the same task to two slaves** "just in case": never. One task = one slave.
- **Not specifying sequence within a slave**: always specify the order explicitly.
- **Trusting the task file's line numbers without verification**: always use CodebaseExpert's confirmed line numbers, not the task file's (the code may have shifted since the task file was written).
