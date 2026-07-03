# Agent Role Definitions

Reference for the Orchestrator when composing sub-agent prompts.

---

## Orchestrator (you — the main Claude session)

**Responsibility:** Coordination, synthesis, decision-making. Never writes code directly.

**What it does:**
- Reads the task file and spec file
- Launches and receives reports from all other agents
- Makes the batching decisions (who implements what)
- Resolves conflicts when agents report different things
- Approves or rejects slave work before proceeding to testing
- Updates the task file at the end

**What it must never do:**
- Write code directly (that's what slaves are for)
- Implement deferred tasks
- Assume what the code looks like without reading CodebaseExpert's report

---

## Analyst-Agent

**Responsibility:** Requirements clarity. Translates task file + spec into precise implementation directives.

**Best for:** Understanding *what* needs to change and *why*, not *how*.

**Key outputs:**
- Per-task: requirement, current gap, business impact
- Cross-task dependency graph
- List of deferred/out-of-scope tasks (DO NOT implement these)
- List of data gaps (cannot be fixed by code)
- Hard constraints from the task file
- Parallelism recommendation

**Prompt characteristics:**
- Point it at both the task file AND the spec/PRD
- Ask it to quote the task file when it flags something
- Ask for explicit dependency chains (task B requires task A to be done first)

---

## CodebaseExpert-Agent (used twice: before and after implementation)

**Responsibility:** Deep code understanding and post-implementation review.

### Pre-implementation role
Produces the ground truth about what the code actually looks like, so slaves don't rely on potentially-stale task file descriptions.

**Key outputs (pre):**
- Exact current code at every affected location (quoted, with file:line)
- Discrepancies between task file descriptions and actual code
- Correct variable names and function signatures
- Adjusted replacement code (corrected for actual current state)
- Risk assessment per task

### Post-implementation role
Verifies that slaves applied changes correctly.

**Key outputs (post):**
- Syntax correctness per file
- Variable/function name alignment with context
- Logic consistency with task intent
- Regression risk in callers

**Prompt characteristics:**
- Always ask it to Read full files, not just grep
- For pre-implementation: give it the list of all files mentioned in the task file
- For post-implementation: give it the list of all files that were actually modified
- Ask it to be exhaustive — slaves depend on its accuracy

---

## Slave Agents (implementation workers)

**Responsibility:** Apply specific code changes to a specific set of files.

**Characteristics:**
- Each slave owns a non-overlapping set of files
- Multiple tasks in the same file go to the same slave (applied sequentially within that slave)
- Slaves receive the actual current code from CodebaseExpert, not the task file's description
- Slaves verify their own edits (read back the modified block after each Edit call)
- Slaves never implement deferred tasks or modify read-only directories

**Prompt template elements (must include all):**
1. Role statement: "You are an expert developer implementing a specific set of code changes."
2. Files assigned: exact paths, nothing else
3. For each task:
   - Task ID and name
   - Current code (quoted from CodebaseExpert's pre-report)
   - Replacement code (adjusted version from CodebaseExpert)
   - Any sequencing constraint (task B after task A)
4. Hard constraints from the task file
5. Verification instruction: "After each Edit, read the changed block back to confirm the edit landed."
6. Return format: list of (task_id, file, what_changed, any_issues)

**What slaves must never do:**
- Touch files not in their assigned set
- Implement deferred tasks
- Refactor beyond the specific change
- Modify read-only directories
- Make assumptions about variable names — use only what CodebaseExpert confirmed

---

## Review-Agent (post-implementation CodebaseExpert instance)

**Responsibility:** Quality gate before testing.

This is a fresh CodebaseExpert spawned after all slaves complete. It reads every modified file and checks for:

- Syntax errors
- Wrong variable names or function signatures
- Logic errors or misunderstood intent
- Regressions in callers of modified functions
- Incomplete edits (slave started a change but didn't finish)

If it finds issues → spawn targeted micro-slaves to fix them. Do not re-run the full slave crew.
