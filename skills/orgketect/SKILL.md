---
name: orgketect
description: Write high-level implementation plans as .org files with context-inheriting tree structure and dependency graph for parallel execution. Use when the user wants to plan a feature, project, or multi-step implementation before coding. Trigger on phrases like "plan this", "write a plan", "let's architect", "break this down", "implementation plan", or when a brainstorming session produces a spec ready for planning. Also trigger when the user mentions org-mode plans, .org planning, or orgketect.
---

# Orgkitect

Write implementation plans as org-mode files. Plans are high-level — they define *what* to build and *why*, structured so executing agents inherit context through the org tree. Implementation details (exact code, TDD steps, error handling patterns) are the executing agent's job.

The org tree is the plan's primary abstraction. Each heading level adds specificity, and properties propagate downward — an executing agent reading the path from root to its task has complete context without reading sibling branches.

**Announce at start:** "Using orgketect to write the implementation plan."

## Plan Location

Every plan is a **directory** under `.org/plans/`. Single-file plans
are not supported — the smallest valid plan is an overview file plus
one milestone file.

```
.org/plans/YYYY-MM-DD-<feature-name>/
  00-overview.org           # goals, architecture, file map, milestone list
  01-<milestone-slug>.org   # first milestone's components + tasks
  02-<milestone-slug>.org
  ...
```

Milestone files are numbered by intended dependency/review order. The
word "milestone" does **not** appear in filenames — just a short slug.

### Bootstrapping a new project

If `.org/` doesn't exist yet, create the plan directory directly:
`mkdir -p .org/plans/YYYY-MM-DD-<feature-name>/`. The project root is
already in org-mcp's allowed directories (it comes in via the MCP
`initialize` roots handshake), so anything under `.org/` is
automatically writable by org-mcp tools — no extra config.

You do **not** need org-mcp's capture tool to *create* files — write
them directly with the file-write tools available in the session.
org-mcp is only needed for assigning org-ids (see below).

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

**Never hand-write `:ID:` properties.** Do not call `uuidgen`, do not
invent slugs, do not write `:ID:` lines. Org-ids must come from
`org-id-get-create` running inside Emacs so they stay consistent with
how orgkestrate and orgket read the plan later.

The flow is **write-then-assign**, not capture-per-task:

1. **Write the full plan file directly** (Write tool, not capture).
   Include every heading, property drawer (minus `:ID:`), body, and
   acceptance criteria. For `:DEPENDS:`, use human-readable
   placeholders that reference sibling task headlines, prefixed with
   `@`:

   ```org
   *** TODO Validate input
   :PROPERTIES:
   :FILES: src/auth/validate.ts
   :END:

   *** TODO Hash password
   :PROPERTIES:
   :FILES: src/auth/hash.ts
   :DEPENDS: @validate-input
   :END:
   ```

   The placeholder is just `@` + a slugified version of the target
   headline. It's a marker, not a real id.

2. **Call `org_assign_ids`** on the file. The tool walks every
   heading carrying a TODO-state keyword (including custom states
   like `NEXT`, `WAITING`) and assigns an org-id via
   `org-id-get-create`. It returns:

   ```
   :file "/path/to/plan.org"
   :entries ((:id "abc-123" :headline "Validate input" :level 3)
             (:id "def-456" :headline "Hash password"   :level 3)
             ...)
   ```

   Non-task headings (`* Plan`, `** Component A`, `** File Structure`)
   are intentionally left without ids — they're organizational scaffold,
   not dispatch targets.

3. **Resolve `@`-placeholders to real ids.** Build a slug → id map
   from the returned entries (slugify each `:headline` the same way
   you slugified placeholders), then use the Edit tool to replace
   each `@slug` reference in `:DEPENDS:` with the corresponding id.

4. **Multi-file plans:** repeat steps 1–3 per file. Cross-file
   `:DEPENDS:` works the same way — reference task ids returned from
   earlier files. Write files in dependency order so upstream ids are
   known when you write downstream files.

#### Why a milestone is not a "group dependency"

`org_assign_ids` only assigns ids to TODO-state headings on purpose.
If you need a milestone gate ("don't start integration tests until
all of Authentication is done"), model it as an explicit gate task:

```org
*** TODO Authentication complete
:PROPERTIES:
:DEPENDS: @validate-input, @hash-password, @issue-token
:END:
```

Now downstream tasks depend on the gate task — no special "group id"
semantics needed in the executor.

If the `org_assign_ids` tool is not available in the current session,
stop and tell the user — do not fall back to `uuidgen` or
hand-written slugs.

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
