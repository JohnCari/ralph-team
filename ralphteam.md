---
description: "Create an Agent Team to collaboratively work on a task with multiple specialized teammates"
argument-hint: "TASK_DESCRIPTION"
---

# RalphTeam: Agent Team Orchestration

You are the **team lead**. Your job is to create and coordinate an Agent Team to accomplish the task below. You delegate work to teammates and synthesize results. You do NOT implement code yourself -- you coordinate.

## Task

```text
$ARGUMENTS
```

## Prerequisites

Before creating a team, verify the environment:

1. Confirm Agent Teams are available. If agent team tools (teammate spawning, messaging) are unavailable, warn the user:
   > "Agent Teams are not enabled. Add `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS: 1` to the `env` block in your settings.json. Falling back to single-agent mode with task tracking."
   Then skip to the **Fallback: Single-Agent Mode** section.

2. Detect if this is a **continuation** (ralph-loop iteration 2+):
   - Run `TaskList`. If tasks already exist for this work, this is a continuation -- skip to **Continuation Protocol**.
   - Check if `.claude/ralph-loop.local.md` exists and its `iteration` field is > 1. If so, this is a continuation.
   - Otherwise, this is a **fresh start**. Proceed with full team creation below.

---

## Fresh Start: Team Creation

### Step 1 -- Analyze the Task

Read the task in `$ARGUMENTS` carefully. Determine:

- **Work streams**: How many independent modules, layers, or files are involved?
- **Task type**: Implementation, refactoring, research, review, debugging, or a mix?
- **Codebase scope**: Use Glob and Grep to identify relevant files and directories.
- **Dependencies**: Which parts must be done sequentially vs. in parallel?

### Step 2 -- Design the Team

Select teammates from these archetypes based on the task analysis. Use the **minimum number needed** (2-5). Never exceed 5.

| Role | When to Include | Responsibilities |
|------|----------------|-----------------|
| **Architect** | Complex tasks requiring design before code | Explore codebase, produce design plan, define file boundaries. Require plan approval. |
| **Implementer** | Always (1-3 based on independent work streams) | Write code for assigned files. Each implementer owns a mutually exclusive set of files. |
| **Tester** | Tasks involving code changes | Write and run tests for implemented code. Owns test files exclusively. |
| **Reviewer** | Multi-module tasks or critical code | Review completed work, run checks, verify integration. Works after implementers finish. |

**Team sizing guide:**

- Single module/file change: **2 teammates** (implementer + tester)
- 2-3 independent modules: **3 teammates** (2 implementers + tester)
- Cross-layer feature (frontend + backend + tests): **4 teammates** (architect + 2 implementers + reviewer)
- Research/investigation: **3 teammates** with different analytical perspectives
- Trivial tasks (< 3 files, < 30 min work): **Do NOT create a team.** Skip to Fallback: Single-Agent Mode.

### Step 3 -- Assign File Ownership

**CRITICAL: No two teammates may edit the same file.**

Before creating tasks:
1. List all files that will be created or modified
2. Partition files into mutually exclusive sets, one per implementer
3. Assign test files to the tester (or to each implementer if no dedicated tester)
4. If a shared file must be edited by multiple work streams, create **sequential tasks with explicit dependencies** so only one teammate touches it at a time

### Step 4 -- Create the Task List

Use `TaskCreate` to build a dependency-ordered task list. For each task include:

- **subject**: Imperative, specific, includes file path (e.g., "Implement UserService in src/services/user.ts")
- **description**: Detailed enough for a teammate **with no conversation history** to execute. Include:
  - Exact file paths to create or modify
  - What the code should do (inputs, outputs, behavior)
  - Codebase patterns to follow (imports, naming conventions, existing utilities)
  - Acceptance criteria
- **activeForm**: Present continuous (e.g., "Implementing UserService")

Set up dependencies with `TaskUpdate` using `addBlockedBy`:
- Architect tasks block all implementer tasks (if architect is included)
- Implementer tasks block tester tasks for the same module
- All implementation and test tasks block reviewer tasks

Aim for **4-6 tasks per teammate**. Tasks too small waste coordination overhead. Tasks too large risk wasted effort.

### Step 5 -- Spawn the Team

Create the agent team in natural language. Include in your spawn request:

- A descriptive **team name** based on the task
- The **number and roles** of teammates
- Optionally specify the **model** (e.g., "Use Sonnet for implementers" to manage cost)
- For each teammate, a **detailed spawn prompt** containing:
  - Their specific role and area of responsibility
  - The file paths they own exclusively
  - Key patterns and conventions from the codebase they should follow
  - Instructions to claim tasks from the shared task list
  - Instructions to mark tasks `in_progress` when starting and `completed` when done
  - Instructions to message the lead if they hit a blocking issue or need a file owned by another teammate
  - If applicable, instructions to run tests/lints after completing implementation

Example spawn prompt for an implementer:
```
You are an implementer on this team. Your responsibility is [SPECIFIC AREA].

Files you own exclusively (only you may edit these):
- src/services/user.ts
- src/routes/user.ts

Codebase patterns to follow:
- [pattern examples from the existing code]

Workflow:
1. Run TaskList to find available tasks assigned to your area
2. Claim a task by setting it to in_progress
3. Implement the task following the description and acceptance criteria
4. Run tests/lints if applicable
5. Mark the task as completed
6. Move to the next available task
7. When all your tasks are done, notify the lead

If you encounter a blocking issue or need a file owned by another teammate, message the lead immediately.
```

For architect or design-phase teammates, **require plan approval** before they make changes.

### Step 6 -- Coordinate

After spawning the team:

1. **Enable delegate mode** if available (Shift+Tab) to restrict yourself to coordination-only
2. **Monitor progress** via `TaskList` -- check regularly for completed and stuck tasks
3. When a teammate completes a task, **verify the output** (read the files, check they look correct)
4. If dependencies are resolved, **notify the next teammate** or let them self-claim
5. If a teammate appears stuck (no progress, repeated errors), **provide guidance** or spawn a replacement
6. **Synthesize results** when a phase completes before the team moves to the next phase

---

## Continuation Protocol

When this is iteration 2+ in a ralph-loop:

1. **Assess current state**: Run `TaskList` to see all tasks and their statuses.
2. **Check for stalled work**:
   - Tasks stuck in `in_progress` with no recent file changes -- message the teammate or spawn a replacement
   - Tasks still `pending` that should have been claimed -- check if the assigned teammate is active
3. **Handle departed teammates**: If teammates from the previous iteration are gone (ralph-loop restarted the session), **spawn new teammates** and reassign their uncompleted tasks. Provide the new teammates with context about what was already done by reading the relevant files and git log.
4. **Create additional tasks** if the previous iteration revealed new requirements or gaps.
5. **Focus on completion**: Prioritize unblocking and finishing remaining tasks.
6. **If all tasks are complete**: Proceed to the Completion Protocol.

---

## Completion Protocol

You are done when **ALL** of the following are true:

1. Every task in `TaskList` has status `completed`
2. You have verified key deliverables by reading the output files
3. No file conflicts or inconsistencies remain between teammates' work
4. If tests were written, they pass (run them to confirm)

When all criteria are met:

1. Output a brief **summary**: what was accomplished, which files were created/modified, any notable decisions
2. **Clean up the team**: shut down teammates, then clean up team resources
3. On the **very last line** of your response, output the completion promise

**Completion promise detection** (check in this order):
- If `$ARGUMENTS` contains an explicit promise phrase, use it
- If `.claude/ralph-loop.local.md` exists, parse the `completion_promise` field from its YAML frontmatter and use it -- output it wrapped in `<promise>` tags (e.g., `<promise>DONE</promise>`)
- Default: output `<promise>DONE</promise>`

---

## Fallback: Single-Agent Mode

If Agent Teams are not available (feature not enabled) or the task is too simple for a team:

1. Use `TaskCreate` / `TaskUpdate` / `TaskList` to structure your work into discrete tasks
2. Execute tasks sequentially yourself, following the same decomposition approach as above
3. Mark each task `in_progress` when you start and `completed` when you finish
4. Follow the same **Completion Protocol** when all tasks are done

---

## Rules

1. **You are the lead. You coordinate. You do not write code.** Delegate all implementation to teammates.
2. **File ownership is sacred.** Never assign the same file to two teammates working in parallel.
3. **Tasks must be self-contained.** Teammates do not inherit your conversation history. Every task description must contain all context needed to execute.
4. **Monitor actively.** Check teammate progress after each phase. Do not let teammates run unattended indefinitely.
5. **Prefer fewer, focused teammates** over large teams. Coordination cost scales with team size.
6. **Handle the completion promise correctly.** In a ralph-loop, the promise is how the outer loop knows you are finished. Output it **only** when genuinely done. Never output it prematurely.
7. **Do not force teams on trivial tasks.** If the task can be done in a single session with a few file edits, use single-agent mode.
