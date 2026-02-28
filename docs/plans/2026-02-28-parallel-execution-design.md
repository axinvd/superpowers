# Parallel Execution & Model Selection Design

**Goal:** Make subagent-driven-development support parallel task execution with scoped workers and per-task model selection, using Claude's built-in plan mode for the research phase.

**Scope:** Personal fork — no backward compatibility concerns.

---

## Overview

Two skills change:

1. **writing-plans** — uses EnterPlanMode for codebase research, produces plans with parallel groups, file lists per task, and model annotations
2. **subagent-driven-development** — becomes a DAG orchestrator that dispatches scoped workers in parallel per group

Three templates change:

- `implementer-prompt.md` → replaced by `scoped-worker-prompt.md`
- `spec-reviewer-prompt.md` → deleted
- `code-quality-reviewer-prompt.md` → deleted

---

## 1. Plan Format

Plans become DAGs organized into parallel groups (execution waves).

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development

**Goal:** [One sentence]
**Architecture:** [2-3 sentences]
**Tech Stack:** [Key technologies]

---

## Group 1 (parallel)

### Task 1.1: Create user model [model: haiku]

**Files:**
- Create: `src/models/user.py`
- Test: `tests/models/test_user.py`

**Depends on:** none

**Step 1: Write the failing test**
...

### Task 1.2: Create auth middleware [model: sonnet]

**Files:**
- Create: `src/middleware/auth.py`
- Test: `tests/middleware/test_auth.py`

**Depends on:** none

**Step 1: ...**

---

## Group 2 (parallel)

### Task 2.1: Add login endpoint [model: sonnet]

**Files:**
- Create: `src/routes/login.py`
- Modify: `src/routes/__init__.py`
- Test: `tests/routes/test_login.py`

**Depends on:** Task 1.1, Task 1.2

**Step 1: ...**
```

### Group construction rules

1. List all tasks with their file lists (Create/Modify/Test)
2. Build a conflict graph: tasks A and B conflict if they share any file in their write lists
3. Tasks that don't conflict → same parallel group
4. Tasks that depend on outputs of earlier tasks → later group
5. When in doubt, serialize. Wrong parallelism is worse than missed parallelism.

### Model annotation guidelines

Each task gets `[model: X]`:

- **haiku**: Single-file changes, config edits, boilerplate, simple tests, renames, moving code without logic changes. Typically <50 lines changed.
- **sonnet**: Feature implementation, refactoring with logic changes, multi-file coordination, writing meaningful tests. Typical development work.
- **opus**: Complex algorithms, subtle correctness requirements, security-sensitive code, architectural decisions within the task, tasks where getting it wrong is expensive.

Default to sonnet when uncertain.

---

## 2. writing-plans Skill Changes

### EnterPlanMode for research

The skill calls `EnterPlanMode` to explore the codebase before writing the plan:

```
Announce "Using writing-plans skill"
  → EnterPlanMode
  → Explore codebase (Glob, Grep, Read)
  → Identify files that will be touched per task
  → Build dependency graph from file overlaps
  → Determine task complexity → assign models
  → Topologically sort into parallel groups
  → Write plan to docs/plans/YYYY-MM-DD-<feature>.md
  → ExitPlanMode (standard plan mode exit, user approves)
```

### Removed: Execution Handoff

No special handoff block. Standard ExitPlanMode behavior. The plan header (`REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development`) ensures any session that picks up the plan uses the right executor.

---

## 3. Scoped Worker Agent

New template `scoped-worker-prompt.md` replaces `implementer-prompt.md`.

### Key constraints

- **File whitelist**: worker may ONLY write to files listed in the task's `Files:` section
- **Read access**: unrestricted (any file for context)
- **No git**: worker does not commit — orchestrator handles all git
- **No project-wide commands**: no formatters, no global linters with `--fix`
- **Out-of-scope escape hatch**: if worker needs changes outside its file list, it stops and reports back

### Prompt template

```
You are implementing Task {N.M}: {task name}

## Your Scope

**You may ONLY write to these files:**
- {explicit file list from plan}

**You may read any file** in the project for context.

### Forbidden
- Writing files outside your scope — if a fix requires out-of-scope changes,
  STOP and return an out-of-scope request
- Any git commands (git add, commit, push, stash, checkout, etc.) —
  orchestrator handles git
- Any command that modifies files outside your scope (formatters, linters
  with --fix on project root, etc.)
- Widening your scope

### If you encounter errors outside your scope
Lint or type-check may report errors outside your file list. **Ignore them.**
Only fix errors in files within your assigned scope.

## Task Description
{FULL TEXT of task from plan}

## Context
{Scene-setting: where this fits, dependencies, prior group results}

## Before You Begin
If anything is unclear — requirements, approach, dependencies — **ask now.**

## Your Job
1. Implement exactly what the task specifies
2. Write tests (TDD)
3. Run tests for your files only
4. Self-review (see below)
5. Report back (do NOT commit)

## Before Reporting Back: Self-Review

Review your work:

**Completeness:**
- Did I implement everything in the spec?
- Are there edge cases I didn't handle?

**Quality:**
- Are names clear and accurate?
- Is the code clean and maintainable?
- Did I follow existing codebase patterns?

**Discipline:**
- Did I avoid overbuilding (YAGNI)?
- Did I only build what was requested?

**Testing:**
- Do tests verify behavior (not just mock behavior)?
- Did I follow TDD?

Fix any issues found before reporting.

## Report Format

When done, report one of:
- **Success**: files modified, what changed, test results, self-review findings
- **Failure**: what went wrong, what you tried, why it didn't work
- **Out-of-scope request**: what change is needed outside your scope, where, why
```

---

## 4. Orchestrator (subagent-driven-development rewrite)

### Flow

```
Read plan
  → Parse groups, tasks, file lists, models
  → Validate no file overlaps within groups
  → For each group:
      ├── Dispatch all tasks in parallel (scoped workers)
      │   - subagent_type: "general-purpose"
      │   - model: haiku/sonnet/opus (from plan)
      │   - prompt: scoped-worker-prompt.md filled in
      ├── Wait for all to complete
      ├── Handle results:
      │   - Success → collect
      │   - Failure → orchestrator diagnoses, re-dispatches or fixes
      │   - Out-of-scope → orchestrator handles directly or defers to next group
      ├── Orchestrator commits: git add {all files} && git commit
      └── Move to next group
  → finishing-a-development-branch
```

### Parallel dispatch

All tasks in a group are dispatched in a single message with multiple `Task` tool calls:

```
Task("Implement Task 1.1: Create user model", model: "haiku", ...)
Task("Implement Task 1.2: Create auth middleware", model: "sonnet", ...)
// Both run concurrently
```

### Commit strategy

One commit per group:
```
feat: group 1 — create user model, auth middleware
```

### Out-of-scope handling

When a worker returns an out-of-scope request:
- If it's a small fix (< 5 lines): orchestrator makes the change directly
- If it's significant: orchestrator creates a new task in the next group

### Error handling

When a worker fails:
- Orchestrator reads the failure report
- Either fixes the issue directly or re-dispatches the worker with additional context
- Does not re-dispatch indefinitely — after 2 failures, orchestrator handles it manually or asks user

---

## 5. Deleted Artifacts

- `skills/subagent-driven-development/implementer-prompt.md` — replaced by `scoped-worker-prompt.md`
- `skills/subagent-driven-development/spec-reviewer-prompt.md` — deleted
- `skills/subagent-driven-development/code-quality-reviewer-prompt.md` — deleted
- All review loop logic in SKILL.md
- "Final review" step
- Execution handoff block from writing-plans

---

## 6. What Stays Unchanged

- Bite-sized task granularity (2-5 min steps with TDD)
- TodoWrite tracking for all tasks
- Worker self-review before reporting
- Question-asking before implementation (workers can ask)
- `finishing-a-development-branch` at the end
- `using-git-worktrees` for workspace isolation
- Plan document location (`docs/plans/YYYY-MM-DD-<feature>.md`)
