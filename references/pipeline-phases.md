# Pipeline Phases — Expanded Checklists

Detailed checklists for each phase of the Grilling Crew pipeline.

---

## Phase 1 — Parallel Intelligence Gathering

**Checklist:**
- [ ] Task file read in full by Orchestrator before launching agents
- [ ] Task type identified: fixes / features / refactors / migrations / mixed
- [ ] Analyst-Agent and CodebaseExpert-Agent spawned in the **same message** (parallel)
- [ ] Analyst-Agent given both the task file AND spec file (if provided)
- [ ] CodebaseExpert-Agent given the complete list of files mentioned in the task file
- [ ] Both agents instructed to return structured reports (not prose summaries)

**Exit criteria:** Both agents have returned reports. Do not proceed until both are complete.

---

## Phase 2 — Synthesis and Approach

**Checklist:**
- [ ] File ownership map built (task → files it touches)
- [ ] Conflict chains identified (tasks sharing files merged into same slave batch)
- [ ] Deferred/out-of-scope tasks flagged (will be documented but NOT implemented)
- [ ] Data gaps flagged (will be documented as info flags but NOT fixed via data)
- [ ] Hard constraints extracted from task file (read-only dirs, API limits, etc.)
- [ ] Slave batches defined (see batching-guide.md)
- [ ] Priority order within each slave defined
- [ ] Implementation plan stated briefly before spawning (2-3 sentences per batch)

**Exit criteria:** Batches are defined. Orchestrator has stated the plan.

---

## Phase 3 — Slave Agent Dispatch

**Checklist:**
- [ ] All independent slave batches spawned in a **single message** (parallel Agent tool calls)
- [ ] Each slave prompt includes: role, file list, actual current code, replacement code, sequence constraints, hard constraints, return format
- [ ] No slave prompt uses task-file line numbers without CodebaseExpert verification
- [ ] Sequenced batches (if any) launched after their dependency batch completes
- [ ] Each slave instructed to read back modified blocks to verify edits

**Exit criteria:** All slaves have returned their reports. Orchestrator reviews each report for completeness.

### Slave Report Review Checklist
For each slave's report:
- [ ] Every assigned task is accounted for (implemented, skipped, or flagged as issue)
- [ ] Files modified match what was assigned (no scope creep)
- [ ] No slave touched a read-only directory
- [ ] No slave implemented a deferred task

---

## Phase 4 — Post-Implementation Review

**Checklist:**
- [ ] Review-Agent spawned with the full list of actually-modified files
- [ ] Review-Agent instructed to Read full files, not just changed sections
- [ ] Review-Agent checks: syntax, variable names, function signatures, logic, caller regressions
- [ ] Review-Agent report received

**If Review-Agent finds issues:**
- [ ] Issues categorized: syntax / logic / naming / incomplete edit
- [ ] Targeted fix-slaves spawned for each issue (one per file conflict group)
- [ ] Fixed files re-reviewed (either by another Review-Agent or by Orchestrator reading the file)

**Exit criteria:** Review-Agent reports PASS or all flagged issues have been resolved.

---

## Phase 5 — Testing and Verification

**Checklist:**
- [ ] Server/process started (if needed by the task type)
- [ ] Startup verified (health check, ping, or equivalent)
- [ ] Each task's verification test run (from task file's test commands)
- [ ] For tasks without explicit tests: smoke test run (invoke the code path, confirm no exception)
- [ ] Results recorded: PASS / PARTIAL / FAIL per task

**If a test fails:**
- [ ] Root cause identified (diagnose first — use diagnosing-bugs vocabulary)
- [ ] Is it a code bug or an environment issue?
- [ ] Targeted fix-slave spawned for code bugs
- [ ] Environment issues flagged for user (Orchestrator cannot fix env problems)
- [ ] Fixed test re-run

**Exit criteria:** All tests either PASS or have a documented reason for PARTIAL/FAIL.

### Task type-specific test approaches

| Task type | Preferred test method |
|---|---|
| API endpoint fix | `curl` against running server, check response body |
| Data pipeline fix | Run pipeline on sample input, inspect output structure |
| Parser/NL fix | Run follow-up query sequence, check inherited context |
| Config alignment | Read the config file, assert expected key=value |
| Routing fix | Invoke the affected entity type, check source_status in response |
| Metric/calculation fix | Compare metric value against known expected range |
| Flag generation | Check `report.flags[]` array for expected flag codes |

---

## Phase 6 — Final Report and Task File Update

**Checklist:**
- [ ] Task file opened for editing
- [ ] Each task status updated (use the status table in SKILL.md)
- [ ] `## Implementation Run Summary` section added at the top
- [ ] Summary includes: date, counts (implemented / deferred / data gaps / failed), files modified, remaining work
- [ ] Task file saved
- [ ] Orchestrator gives final summary to user: what was done, what wasn't, what needs follow-up

**Summary format for user:**

```
## Grilling Crew — Run Complete

**Tasks implemented:** X/Y
**Deferred:** N (as specified in task file)
**Data gaps:** M (documented as info flags)
**Failed:** K — [brief reason]

**Files modified:** [list]

**Next steps (if any):** [specific actions the user should take]
```

---

## Phase Decision Tree — When to Pause vs Continue

The Orchestrator should proceed autonomously through all phases. Pause and ask the user only when:

1. **Task file is missing or unreadable** → ask for correct path
2. **Task file contains ambiguous tasks** that cannot be implemented without a product decision → ask the specific question, list the options
3. **A test fails and the root cause is environmental** (missing dependency, wrong credentials, external service down) → report the issue, ask user to fix the environment
4. **A slave reports a conflict** that changes the scope of another task → present the conflict and ask which task takes priority
5. **Review-Agent finds an issue that could be fixed multiple ways** and the choice has significant architectural impact → present the options briefly, ask for preference

In all other cases: keep going.
