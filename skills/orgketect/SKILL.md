---
name: orgketect
description: Write high-level implementation plans as .org files with context-inheriting tree structure and dependency graph for parallel execution. Use when the user wants to plan a feature, project, or multi-step implementation before coding. Trigger on phrases like "plan this", "write a plan", "let's architect", "break this down", "implementation plan", or when a brainstorming session produces a spec ready for planning. Also trigger when the user mentions org-mode plans, .org planning, or orgketect.
---

# Orgkitect

Write implementation plans as org-mode files. Plans are high-level — they define *what* to build and *why*, structured so executing agents inherit context through the org tree. Implementation details (exact code, TDD steps, error handling patterns) are the executing agent's job.

The org tree is the plan's primary abstraction. Each heading level adds specificity, and properties propagate downward — an executing agent reading the path from root to its task has complete context without reading sibling branches.

**Announce at start:** "Using orgketect to write the implementation plan."

## Plan Location

Save plans to `.org/plans/` in the project root:

```
.org/plans/YYYY-MM-DD-<feature-name>.org
```

If `.org/` doesn't exist yet, create it and remind the user to add it to their org-mcp allowed directories so executing agents can query the plan:

```elisp
(add-to-list 'org-mcp-allowed-directories "<project-root>/.org/")
```

## Scope Check

If the spec covers multiple independent subsystems, suggest breaking it into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## Plan Structure

### Header

```org
#+title: <Feature Name> Implementation Plan
#+date: <YYYY-MM-DD>
#+property: GOAL <one sentence describing what this builds>
#+property: TECH_STACK <key technologies/libraries>

* <Feature Name>
:PROPERTIES:
:ARCHITECTURE: <2-3 sentences about the structural approach>
:END:

<Brief description of the feature and its motivation. This is the root
context — every heading below inherits it.>
```

`#+property` lines are file-level and inherited by all headings. The top heading's `:ARCHITECTURE:` property captures structural decisions.

### File Structure (Level 2)

Before defining tasks, map out which files will be created or modified. This scopes context — executing agents see only the files relevant to this feature, not the entire codebase.

```org
** File Structure
:PROPERTIES:
:PURPOSE: Decomposition map for this feature
:END:

| File | Status | Responsibility |
|------+--------+----------------|
| src/auth/handler.ts | create | Request authentication and token validation |
| src/auth/types.ts | create | Shared type definitions for auth module |
| src/middleware.ts | modify | Register auth middleware in the chain |
```

This informs task decomposition. Each task references files from this table.

### Component Headings (Level 2)

Group tasks by component or module. Each component heading carries context its child tasks inherit:

```org
** <Component Name>
:PROPERTIES:
:PURPOSE: <what this component does and why>
:INTERFACES: <inputs, outputs, dependencies on other components>
:END:

<Additional context about this component. An executing agent reading
only this heading and the root should understand where this component
fits in the system.>
```

### Task Headings (Level 3)

Each task is a unit of work one executing agent picks up. It should result in roughly one commit or a small PR (under ~400 lines of change). A task heading plus its inherited context (root → component → task) is self-contained.

```org
*** TODO <Verb> <thing> — <why>
:PROPERTIES:
:FILES: <exact paths to create or modify, comma-separated>
:DEPENDS: <comma-separated org-ids of prerequisite tasks, filled in after capture>
:END:

<What to build and why. Describe the behavior, not the implementation.
Mention edge cases the executing agent should consider. Reference
existing code patterns or docs the agent should look at.>

**** Acceptance Criteria
- <Observable outcome — verifiable by running code or tests>
- <Another observable outcome>

**** What to Test
- <Behavior to cover>
- <Edge case worth a test>
- <Integration point to verify>
```

### What Goes in a Task

- **Files**: Exact paths. File decisions are architectural.
- **Acceptance criteria**: Observable outcomes. "Returns 404 for unknown IDs" not "add error handling."
- **Behavioral description**: What it does, not how. "Validates input against the schema" not "use Joi with a try/catch."
- **Context pointers**: Existing code patterns to follow, docs to read, related components.
- **Test guidance**: Behaviors and edge cases to test, not test code.
- **Dependencies**: Org-ids of tasks that must complete first. Tasks without dependencies can run in parallel.

### Task IDs

**Do not pre-generate task IDs.** Do not call `uuidgen`, do not invent slugs, do not write `:ID:` properties by hand. Org-ids are assigned by org-mcp when the task is captured — the capture tool returns the new id.

Workflow:

1. Draft the plan structure (headlines, files, bodies, acceptance criteria) without `:ID:` or `:DEPENDS:` values.
2. Capture tasks into the plan file via org-mcp's capture/mutate tool, which calls `org-id-get-create` and returns the assigned id.
3. As ids come back, record them and fill in `:DEPENDS:` on dependent tasks in subsequent capture calls (or via a follow-up property update).

If org-mcp tools are not available in the current session, stop and tell the user — do not fall back to `uuidgen` or hand-written slugs. The point of routing through org-mcp is that ids, file writes, and dependency wiring stay consistent with how orgkestrate and orgket read the plan later.

### What Does NOT Go in a Task

- Exact code or pseudocode
- Step-by-step implementation instructions
- TDD red/green/refactor sequences
- Specific error handling or validation patterns
- Library API calls
- Commit messages

The executing agent is skilled. Give it the *what* and *why*; it figures out the *how*.

## Dependency Graph

Dependencies enable parallel execution. Tasks with no unmet dependencies can be dispatched simultaneously by the orchestrator (orgkestrate).

Design the graph intentionally:

- **Maximize parallelism**: If two tasks don't share files or depend on each other's output, they should be independent. Don't add false dependencies.
- **Minimize coupling**: If a task depends on another, it should depend on that task's *output* (an interface, a type, a created file), not its internal implementation. State what the dependency provides in the task description.
- **Keep chains short**: Long sequential chains bottleneck execution. If a chain has 5+ tasks, look for ways to restructure so some can run in parallel with a shared dependency.

## Context Inheritance

Design the tree so context flows downward without repetition:

1. **Root**: Project-wide decisions (goal, architecture, tech stack)
2. **Component**: Module-specific context (purpose, interfaces, boundaries)
3. **Task**: Only task-specific details (files, acceptance, behavior)

A task should never repeat information from its ancestors. If you're writing the same context in multiple tasks, move it up to the component or root level.

## Task Sizing

Each task should be roughly one commit or a small PR:
- Under ~400 lines of change
- Touches a focused set of files
- Produces a testable, working increment

If a task feels too big, split it. More small tasks means better parallelism and more useful execution summaries for dependent tasks.

## Self-Review

After writing the plan, review with fresh eyes:

1. **Spec coverage**: Point to a task for each requirement. List gaps.
2. **Context sufficiency**: Pick any leaf task. Read only root → component → task. Does the executing agent have enough to start? If not, add context to the appropriate ancestor.
3. **Dependency accuracy**: For each `:DEPENDS:`, verify the dependency is real. Remove false dependencies that limit parallelism.
4. **Parallelism check**: Visualize the DAG. Are there tasks that could run in parallel but have unnecessary sequential dependencies?
5. **Task sizing**: Flag any task that looks like it would exceed ~400 lines.
6. **Acceptance concreteness**: Every criterion must be verifiable by running code or tests. Rewrite vague ones.

## Execution Handoff

After saving the plan:

"Plan saved to `.org/plans/<filename>.org`. Use orgkestrate to execute — it walks the dependency graph, dispatches orgket workers in parallel, and feeds execution summaries to downstream agents."

If org-mcp isn't configured for `.org/`, remind the user to add it to `org-mcp-allowed-directories`.
