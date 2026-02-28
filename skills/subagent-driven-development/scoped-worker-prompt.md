# Scoped Worker Prompt Template

Use this template when dispatching a scoped worker subagent for a task within a parallel group.

```
Task tool (general-purpose):
  description: "Implement Task {N.M}: {task name}"
  model: {haiku|sonnet|opus from plan annotation}
  prompt: |
    You are implementing Task {N.M}: {task name}

    ## Your Scope

    **You may ONLY write to these files:**
    - {explicit file list from plan's Files: section}

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

    {FULL TEXT of task from plan — paste it here, don't make subagent read file}

    ## Context

    {Scene-setting: where this fits in the project, what prior groups produced,
     architectural context the worker needs}

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
