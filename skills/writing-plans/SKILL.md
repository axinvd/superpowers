---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for the codebase. Document everything: which files to touch, code, testing, docs to check, how to verify. Bite-sized tasks, organized into parallel groups. DRY. YAGNI. TDD. Frequent commits.

Assume the executor is a skilled developer but knows almost nothing about the toolset or problem domain.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** Invoked by brainstorming skill after design approval. Plan is written to the standard plan mode file (`.claude/plans/*.md`).

## Process

1. Call `EnterPlanMode`
2. Explore the codebase (Glob, Grep, Read) — understand architecture, existing patterns, files that will be touched
3. Break work into tasks with explicit file lists per task
4. Detect file overlaps between tasks — conflicting tasks go in separate sequential groups
5. Annotate each task with model (haiku/sonnet/opus) based on complexity
6. Topologically sort tasks into parallel groups (execution waves)
7. Write plan document
8. Call `ExitPlanMode` — standard plan mode exit, user approves

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** NEXT STEP: Invoke `superpowers:subagent-driven-development` to execute this implementation plan.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Parallel Groups

Tasks are organized into groups (execution waves). All tasks within a group run in parallel. Groups execute sequentially.

```markdown
## Group 1 (parallel)

### Task 1.1: Create user model [model: haiku]

**Files:**
- Create: `src/models/user.py`
- Test: `tests/models/test_user.py`

**Depends on:** none

[steps...]

### Task 1.2: Create auth middleware [model: sonnet]

**Files:**
- Create: `src/middleware/auth.py`
- Test: `tests/middleware/test_auth.py`

**Depends on:** none

[steps...]

---

## Group 2 (parallel)

### Task 2.1: Add login endpoint [model: sonnet]

**Files:**
- Modify: `src/routes/login.py`
- Test: `tests/routes/test_login.py`

**Depends on:** Task 1.1, Task 1.2

[steps...]
```

### Building Parallel Groups

1. List all tasks with their file lists (Create/Modify/Test)
2. Build a conflict graph: tasks A and B conflict if they share any file in their write lists
3. Tasks that don't conflict with each other → same parallel group
4. Tasks that depend on outputs of earlier tasks → later group
5. Within a group, order doesn't matter
6. **When in doubt, serialize.** Wrong parallelism is worse than missed parallelism.

### Assigning Models

Annotate each task with `[model: X]`:

- **haiku**: Single-file changes, config edits, boilerplate, simple tests, renames, moving code without logic changes. Typically <50 lines changed.
- **sonnet**: Feature implementation, refactoring with logic changes, multi-file coordination, writing meaningful tests. Typical development work.
- **opus**: Complex algorithms, subtle correctness requirements, security-sensitive code, architectural decisions within the task, tasks where getting it wrong is expensive.

Default to sonnet when uncertain.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" — step
- "Run it to make sure it fails" — step
- "Implement the minimal code to make the test pass" — step
- "Run the tests and make sure they pass" — step

## Task Structure

````markdown
### Task N.M: [Component Name] [model: sonnet]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Depends on:** Task 1.1 | none

**Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

**Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS
````

## Remember

- Exact file paths always
- Complete code in plan (not "add validation")
- Exact commands with expected output
- Reference relevant skills with @ syntax
- DRY, YAGNI, TDD
- **No commit steps in tasks** — orchestrator commits per group
- **File lists are critical** — they define worker scope and parallel group boundaries
