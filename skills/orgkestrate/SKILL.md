---
name: orgkestrate
description: Execute orgketect implementation plans by walking the dependency DAG, dispatching orgket workers in parallel for independent tasks, and tracking progress via org-mcp. Use when the user wants to execute an orgketect plan, start implementing from a .org plan, or says "execute the plan", "start implementing", "run the plan", "orgkestrate", or after an orgketect plan is written and the user wants to proceed.
---

# Orgkestrate

Execute an orgketect plan by walking its dependency DAG. Independent tasks run in parallel via orgket workers. Each worker reads only its context chain, implements the work, and writes an execution summary back to the org entry. Downstream workers read upstream summaries before starting, so context flows through the plan as work progresses.

**Announce at start:** "Using orgkestrate to execute the plan."

## Setup

### Locate the Plan

If the user doesn't specify a plan file, look in `.org/plans/` for the most recent one. Confirm with the user before proceeding.

### Verify org-mcp Access

The plan's `.org/` directory must be in `org-mcp-allowed-directories`. If org-mcp tools aren't available, fall back to reading/writing .org files directly — but note this loses property inheritance queries. Suggest the user configure org-mcp for the best experience.

### Build the DAG

Read the plan and build a dependency graph from `:DEPENDS:` properties. Identify the initial set of ready tasks (those with no dependencies). Present the DAG to the user before starting:

```
Ready now:     Task A (auth types), Task B (config schema)
After A:       Task C (auth handler)
After A and B: Task D (middleware integration)
After C and D: Task E (API routes)
```

Get user confirmation before dispatching.

## Dispatch Loop

```
while unfinished tasks exist:
  1. Query plan for all TODO tasks
  2. For each, check if all :DEPENDS: are DONE
  3. Dispatch ready tasks as parallel orgket workers
  4. As workers complete:
     a. Read execution summary
     b. Verify summary is present and coherent
     c. Mark task DONE (via org_set_state or direct edit)
     d. Check what new tasks are now unblocked
     e. If execution diverged from plan, pause and consult user
  5. Dispatch newly ready tasks
  6. Report progress
```

### Dispatching an Orgket Worker

Each worker gets a focused prompt assembled from the org tree. The worker does NOT read the entire plan — only its context chain.

**Build the worker prompt from:**

1. **Project context** — root heading body + inherited properties (GOAL, TECH_STACK, ARCHITECTURE)
2. **File structure** — the file structure table from the plan
3. **Component context** — the component heading body + properties (PURPOSE, INTERFACES)
4. **Task details** — the task heading, properties (FILES, ACCEPTANCE), body, acceptance criteria, and test guidance
5. **Upstream summaries** — execution summaries from completed dependency tasks

If org-mcp is available, use `org_get_entry` and `org_get_properties` (with inherit) to assemble this context. Otherwise, read the .org file and extract the relevant headings.

Dispatch the worker as a subagent with the assembled prompt. See the orgket skill for the worker's responsibilities.

### Reading Upstream Context

Before dispatching a task with dependencies, read execution summaries from completed dependency tasks. These summaries capture what *actually* happened — interface decisions, file paths that changed, surprises encountered. Include them in the worker prompt so downstream agents build on reality, not just the plan.

## Error Handling

### Worker Fails

If a worker fails or produces broken output:
- Do NOT automatically retry
- Report to the user: which task, what went wrong, the worker's output
- Let the user decide: retry, modify the plan, or fix manually

### Dependency Conflict

If a completed task's summary contradicts what a downstream task expects (renamed function, changed file path, different interface):
- Pause before dispatching the downstream task
- Show the user the conflict
- Options: update the downstream task in the plan, or proceed and let the worker adapt

### Stuck DAG

If all remaining tasks are blocked (circular dependency or failed prerequisite):
- Report which tasks are blocked and why
- Suggest fixes (remove circular dep, manually complete the failed prereq)

## Progress Reporting

After each dispatch/completion cycle, brief status update:

```
Completed: Task A (auth types), Task B (config schema)
Running:   Task C (auth handler)
Ready:     Task D (middleware) — waiting for C
Blocked:   Task E (API routes) — waiting for C, D
```

## Post-Execution

When all tasks are DONE:

1. Report final status — all tasks with their summaries
2. Summarize any divergences from the original plan
3. The completed `.org` file is now a record of what was planned vs. built — suggest the user review it
4. If new patterns or conventions emerged, suggest adding them to CLAUDE.md
