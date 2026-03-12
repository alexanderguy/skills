---
name: dispatch
argument-hint: <goal-or-plan-file>
description: Orchestrate parallel subagent execution with DAG scheduling, fan-out/fan-in, and iterative fix loops
---

# Dispatch

Orchestrate parallel subagent execution across a dependency graph. Fan out work to subagents, fan in results, validate, critique, verify, fix failures, commit, and repeat until done.

## Initialization

Before doing anything else:

1. Load the `style` and `philosophy` skills
2. Detect what `$ARGUMENTS` is (the argument string passed to the skill by the caller):
   - If it resolves to an existing `dispatch/<name>/dispatch.yaml`, resume that run (skip to Resumability section)
   - If it resolves to an existing file, treat it as a plan to execute (skip to Phase 2)
   - Otherwise, treat it as a goal description and enter Phase 1

## Phase 1: Planning

When `$ARGUMENTS` is a goal description:

0. **Validate requirements completeness:**
   - Confirm the goal is sufficiently specified to create implementable tasks
   - Verify you have: clear objective, functional requirements, constraints, edge cases
   - **If requirements are vague, incomplete, or contradictory: STOP and return to caller**
   - Do not proceed with task breakdown until requirements are actionable and complete
   - When in doubt, ask: "Can an implementation agent succeed with only this information?"

0b. **Capture baseline build state:**
   - Before creating any tasks, run a full build to establish baseline
   - Consult project build documentation (README.md, CONTRIBUTING.md, Makefile, package.json scripts, etc.) to determine build commands
   - Full build typically includes: generate (if applicable) → lint → build → test
   - Save output to `dispatch/<run-name>/baseline-build.log`
   - This baseline will be compared in Phase 5 to detect regressions

1. Use `explore` agents to understand the codebase and scope of work
2. **Extract quality gates from loaded skills:**
   - From `style`: Per-task build verification, per-task test verification for test tasks, never work around failing builds
   - From `philosophy`: Tests as first-class verification, commits that enable debugging
   - From project-specific docs (CONVENTIONS.md, DEV.md, etc.): Project requirements
   - Create a checklist of requirements for this specific plan
3. Break the goal into discrete tasks, each small enough for a single subagent
4. Identify dependencies between tasks (DAG edges)
5. Select the right agent type for each task:
   - `explore` for research and analysis (read-only)
   - `intern` for well-defined implementation tasks (clear requirements, straightforward changes, mechanical work)
   - `general` for complex implementation requiring judgment (architectural decisions, open-ended problems, novel solutions)
6. Detect project verification commands from package.json, Makefile, or similar
7. **Add per-task verification to task plans:**
   - Tasks that produce compiled code (C, Rust, TypeScript, etc.): add build command to verify compilation
   - Tasks that write tests: add test command to verify the tests actually pass
   - Tasks that modify critical paths: add relevant subset of test suite
   - This catches failures early, at the task level, rather than discovering 15 broken tasks at Phase 5
8. **Choose commit strategy:**
   - Use `per-task` when debuggability is critical (enables bisection, isolates failures) - **this is the default**
   - Use `grouped` only when history cleanliness matters AND comprehensive Phase 5 verification will catch all issues
   - Grouped commits make debugging harder: if Phase 5 fails, you can't tell which of 5 bundled tasks broke it
9. **Decide which tasks need critique:**
   - Complex tasks (general agent, complicated objectives) → critique enabled
   - Medium complexity intern tasks → critique enabled
   - Simple intern tasks → critique disabled (trust self-report)
   - When unsure, consult greybeard about complexity
   - Mark in manifest with `critique.enabled: true/false` per task
10. **Verify plan against quality gates checklist** - ensure requirements from loaded skills are satisfied
11. **Present quality gate verification to user** before presenting the full plan (see template below)
    - Include which tasks will be critiqued
    - User can request additional tasks be critiqued if they see something missed
12. Create the dispatch directory structure, manifest, and task files (Phase 2)

### Agent Type Selection Guide

#### Task Agent Types

Use `intern` when:
- The task has a detailed implementation plan with specific file changes
- The approach is clear and mechanical (threading a parameter through a stack, updating all call sites, etc.)
- Success criteria are unambiguous (tests pass, lint passes)
- No architectural decisions are needed

Use `general` when:
- The task requires making technical trade-offs
- The implementation approach is not fully specified
- The task involves designing new abstractions or patterns
- The scope may need adjustment based on what's discovered

Use `explore` when:
- The task is pure research (finding patterns, understanding structure)
- No code changes are required
- The output is findings for downstream tasks to consume

#### Critique Agent Types

Use `critique` (default):
- Specialized code review focused on correctness, completeness, and adherence to requirements
- Verifies objectives are met, checks constraints, and identifies issues without fixing them
- This is the recommended agent type for all critique operations

### Quality Gates Verification Template

Before presenting any dispatch plan to the user, verify it against the loaded skills and present this checklist:

```markdown
## Quality Gate Verification

From `style` skill:
[ ] All code-producing tasks have build verification
[ ] All test-writing tasks verify tests pass
[ ] No workarounds for failing builds

From `philosophy` skill:
[ ] Tests treated as first-class verification
[ ] Commit strategy enables debugging

From AGENTS.md:
[ ] If Phase 5 fails, can isolate which task caused it
[ ] Plan doesn't require debugging multiple tasks simultaneously
[ ] Fixes happen at the right layer (not symptom-chasing)

From project-specific docs:
[ ] [Project-specific requirements if any]

Status:
✅ [Requirement met]: [evidence from plan]
✅ [Requirement met]: [evidence from plan]
❌ [Gap found]: [how you addressed it]

Commit strategy rationale:
[Explain why per-task vs grouped, referencing debuggability vs history cleanliness]
```

Present this verification to the user BEFORE presenting the full dispatch plan. This ensures quality requirements are addressed proactively, not discovered during execution.

**Why this matters:**

Late failure detection is expensive. If 15 tasks complete and Phase 5 verification fails, you must debug all 15 tasks to find the culprit. Grouped commits hide boundaries, making bisection impossible.

Early failure detection is cheap. If task 3a writes tests and immediately runs them, you know task 3a is broken and fix it in isolation.

The quality gates prevent the expensive scenario.

When `$ARGUMENTS` is a plan file:

1. Read the plan and extract tasks, dependencies, and verification steps
2. Create the dispatch directory structure from the plan

Regardless of input type, present the plan to the user for approval before executing. Present the quality gate verification first, then a summary of all tasks, their dependencies, agent types, and the verification commands. The user may approve as-is, request modifications (add/remove/reorder tasks, change dependencies), or abort.

## Phase 2: Directory Structure

Create the following structure:

```
dispatch/
  <run-name>/
    dispatch.yaml
    1a-extract_auth_module/
      plan.md
    1b-extract_logging_module/
      plan.md
    2a-integrate_modules/
      plan.md
    2b-update_shared_middleware/
      plan.md
    3a-cleanup_legacy_imports/
      plan.md
```

The `<run-name>` should be a short kebab-case description of the goal (e.g., `migrate-api-routes`, `extract-shared-modules`). The `dispatch/` directory should be added to `.gitignore` to keep orchestration artifacts out of the project's commit history.

### Task directory naming

Task directories are named so a human scanning the directory can immediately understand execution order and purpose. The format is:

```
<level><sequence>-<short_description>
```

- **Level number** (1, 2, 3...): Derived from the DAG depth. Tasks with no dependencies are level 1. Tasks whose dependencies are all in level 1 are level 2. The level is the longest path from any root to that task, plus one.
- **Sequence letter** (a, b, c...): Distinguishes tasks within the same level. Tasks at the same level can run in parallel.
- **Separator**: A single hyphen between the prefix and the description.
- **Description**: Short, underscore-separated, human-readable. Derived from the task objective. Underscores (not hyphens) are used so that the single hyphen between the prefix and description is visually unambiguous.

This naming means:
- `1a` and `1b` are obviously parallel (same level)
- `2a` obviously comes after level 1
- The description tells you what each task does without opening the file

The directory name is also the task's `id` in `dispatch.yaml`.

#### Fix task naming

Fix tasks (created by critique rejection or verification failure) use a different format to distinguish them from planned tasks:

```
<original-level><original-sequence>-fix<n>-<short_description>
```

For example, if task `1a-extract_auth_module` fails critique, the fix task might be `1a-fix1-resolve_critique_issues`. A second fix attempt would be `1a-fix2-handle_error_edge_case`.

The orchestrator determines `<n>` by counting existing fix tasks for the same original in the manifest and adding one.

This naming:
- Sorts fix tasks adjacent to the original task in directory listings
- Preserves the original level/sequence prefix so it's clear what is being fixed
- The `-fix<n>-` segment clearly marks it as a fix, not a planned task
- Does not pollute the level numbering of planned tasks

Each task directory is the subagent's scratchpad. It reads its plan from there, does its work, and writes its results there. After execution, a task directory looks like:

```
1a-extract_auth_module/
  plan.md              # input: instructions for the subagent
  output.yaml          # output: structured results (source of truth)
  critique.yaml        # output: critique verdict (if critique enabled)
  ...                  # scratch: any other files the subagent creates
```

### Manifest format

The central manifest is `dispatch.yaml`:

```yaml
goal: "Short description of the overall goal"
status: pending              # see "Run status transitions" below
max-parallel: 5              # default 5; total concurrent subagents including critique agents
created: YYYY-MM-DD          # informational; not used by the execution engine

verify:
  workdir: ""                # working directory for commands; empty or omitted = repo root
  build: "bun run build"     # omit if not applicable
  test: "bun test"           # omit if not applicable
  lint: "bun run lint"       # omit if not applicable
  custom:                    # list of {name, command} objects
    - name: integration-tests
      command: "bun test:integration"

critique:
  enabled: true              # global default; tasks can override
  agent: critique            # defaults to critique; must be an agent that can write critique.yaml

commits:
  strategy: per-task         # per-task | single | grouped
  approval: ask-once         # ask-once | ask-each | auto
  prefix: ""                 # optional prefix for commit messages
  message-source: objective  # objective (from plan.md) | notes (from output.yaml)

results:                       # populated by Phase 6; empty until then
  commits: []                  # list of {sha, message, files, tasks} objects

unexpected-modifications: ask  # ask | accept

deviation-handling:
  threshold: moderate         # escalate at this severity and above (minor | moderate | major)
  auto-fix: true              # Karen can create fix tasks below threshold
  escalate-to: user           # user | greybeard (who to consult for threshold+ deviations)

tasks:
  - id: 1a-extract_auth_module
    agent: general           # general | explore
    depends-on: []           # task IDs that must complete first
    receives: []             # task IDs whose output to inject (defaults to depends-on)
    status: pending          # pending | dispatched | completed | failed | fixing
    critique:                # optional per-task override
      enabled: true
      agent: critique        # defaults to critique
      prompt: |
        Custom critique instructions for this task.
    commit-group: ""         # optional; used with 'grouped' commit strategy

  - id: 2a-integrate_modules
    agent: general
    depends-on: [1a-extract_auth_module, 1b-extract_logging_module]
    receives: [1a-extract_auth_module]  # only needs auth output; depends on logging for ordering only
    status: pending
```

### Run status transitions

| Status | When |
|---|---|
| `pending` | Manifest created, plan not yet validated or approved |
| `in-progress` | Execution has started (entering Phase 4) |
| `completed` | All tasks completed, verification passed, commits done |
| `failed` | Orchestrator cannot continue: unresolvable deadlock, user chose to abort, or all remaining tasks have failed with no fix path |

### Repository root

The repository root is the nearest ancestor directory containing `.git`. The orchestrator determines this once during initialization and uses it for:
- Telling subagents where they are working
- Running verification commands
- Resolving relative paths in `files-modified`

### Task fields

| Field | Required | Description |
|---|---|---|
| `id` | yes | Directory name. Format: `<level><sequence>-<description>`. Fix tasks use `<level><sequence>-fix<n>-<description>`. |
| `agent` | yes | Subagent type: `general` or `explore` |
| `depends-on` | yes | Task IDs that must reach `completed` before this task starts (empty list if none). Exception: a fix task may proceed when the specific dependency it fixes is `fixing` -- see Step 1 for the precise rule. |
| `receives` | no | Task IDs whose `output.yaml` to inject as context. Defaults to `depends-on`. Must be a subset of `depends-on`. Use to narrow injection when a task depends on many upstream tasks but only needs context from some. |
| `fixes` | no | The task ID (or list of task IDs) this fix task is correcting. Only present on fix tasks. Used by the execution engine to transition each referenced original task from `fixing` to `completed` when the fix succeeds. Always references original planned tasks, not previous fix tasks. |
| `status` | yes | Current execution status |
| `critique` | no | Per-task critique config; overrides the global `critique` setting |
| `commit-group` | no | Group name for the `grouped` commit strategy. Fix tasks inherit this from the task they fix. |

### Task status values

| Status | Meaning |
|---|---|
| `pending` | Not yet ready to dispatch (dependencies incomplete) |
| `dispatched` | Currently being executed by a subagent |
| `completed` | Finished successfully (`output.yaml` confirms, and critique passed if enabled) |
| `failed` | Failed after dispatch (`output.yaml` reports failure or is missing) |
| `fixing` | A fix task has been created for this task. The task ran and produced output, but that output needs correction. `fixing` satisfies `depends-on` only for fix tasks that target this specific task via `fixes` (see Step 1 for the precise rule). Regular downstream tasks must wait for `completed`. The execution engine transitions this to `completed` when a fix task with this task in its `fixes` field succeeds. |

### Task file format

Each task's `plan.md`. The YAML frontmatter duplicates fields from `dispatch.yaml` for human readability. If they diverge, `dispatch.yaml` is authoritative -- the execution engine never reads `plan.md` frontmatter:

```markdown
---
id: 1a-extract_auth_module
depends-on: []
agent: general
---

## Objective

What the subagent should accomplish.

## Requirements Covered

Link to specific requirements this task implements (from spec file or user request):
- Requirement 1: Brief description
- Requirement 2: Brief description

This ensures traceability and verifies all requirements are addressed by the plan.

## Context

Relevant codebase context, file locations, patterns to follow.

## Files to Modify

- `path/to/file.ts` - what to change

## Constraints

- Rules the subagent must follow
- Files it must NOT touch
- Patterns it must adhere to

## Verification

How you will demonstrate this task is complete and correct:

**Level:** [automated | manual | review]

**Automated:** (for functional changes with testable outcomes)
- Commands: List commands you will run (refer to project build docs for correct commands)
- Expected: What success looks like (e.g., "all tests pass", "builds without errors")
- Evidence: Full command output will be captured to `verification.log`

**Manual:** (for changes requiring demonstration)
- Steps: How you will verify it works (describe the manual process)
- Evidence: Command output, logs, or observations captured to `manual-evidence.log`
- Why manual: Explain why automated verification isn't applicable

**Review:** (for structural changes - refactors, moves, renames)
- Compile check: Command to verify no syntax errors (from project build docs)
- Lint check: Project lint command (from project build docs)
- Unit tests: Run tests for modified files if they exist (from project build docs)
- Evidence: Output captured to `verification.log`
- Notes: What changed and why review level was chosen

**Evidence Requirements:**

Before marking this task complete, you MUST:
1. Execute the verification steps defined above
2. Capture full output to evidence files in your task directory
3. Write summary to output.yaml with references to evidence files

Do NOT claim completion without running verification and capturing evidence.

## Deviation Reporting

**CRITICAL:** You MUST report ANY deviation from this plan, even minor ones.

**Deviations to report:**
- Modified files NOT listed in "Files to Modify"
- Changed implementation approach from what was described
- Scope changes (did more or less than planned)
- Broke constraints (touched forbidden files, violated patterns)
- Changed verification approach
- Any other way you deviated from the plan

**How to report:**
Add `deviations` field to output.yaml:

```yaml
deviations:
  - type: files_not_in_plan | approach_changed | scope_changed | constraint_violation | verification_changed
    description: "Exactly what deviated from the plan and why"
    severity: minor | moderate | major
    justification: "Why the deviation was necessary or beneficial"
```

**Severity levels:**
- **minor:** Cosmetic changes, logging improvements, documentation fixes, typo corrections
- **moderate:** Different file than planned, slightly different approach, minor scope change
- **major:** Architecture change, violated constraints, significant scope change

**If NO deviations:** Include `deviations: []` to confirm you followed the plan exactly.

**Consequence:** If deviations are detected in Step 4 that you didn't report here, your task will be marked FAILED for dishonesty.

## Upstream Context

(This section is empty in the plan file on disk. At dispatch time, the
orchestrator injects upstream output.yaml content into the subagent's prompt
directly -- plan.md is not modified.)

## Output Contract

Before finishing, you MUST write `output.yaml` to your task directory.
The orchestrator will tell you the repository root and your task directory
path when dispatching you.

Required fields:

- `status`: completed | failed
- `files-modified`: list of files you changed (paths relative to repo root)
- `verification-summary`:
  - `level`: automated | manual | review
  - `evidence-files`: list of evidence files in task directory (e.g., [`verification.log`, `test-results.json`])
  - `result`: brief summary (e.g., "all 42 tests passed", "server responded with 200")
- `deviations`: list of deviations from plan (empty list `[]` if none)
  - `type`: files_not_in_plan | approach_changed | scope_changed | constraint_violation | verification_changed
  - `description`: what deviated from the plan
  - `severity`: minor | moderate | major
  - `justification`: why the deviation was necessary
- `exports`: structured data downstream tasks may need (object, can be empty)
- `notes`: free-text context for the orchestrator and downstream tasks
- `error`: if status is failed, describe what went wrong; omit or leave empty when status is completed

Example:

    status: completed
    files-modified:
      - src/routes/users/list.ts
      - src/routes/users/create.ts
    verification-summary:
      level: automated
      evidence-files:
        - verification.log
        - test-results.json
      result: "all tests passed (15/15), build successful"
    deviations:
      - type: files_not_in_plan
        description: "Also modified src/utils/validation.ts to add shared validation helper"
        severity: minor
        justification: "Both routes needed the same email validation logic, extracted to avoid duplication"
    exports:
      handler-pattern: defineHandler
      breaking-changes: false
    notes: |
      Migrated both routes. The create route had a custom error
      handler that was refactored into the middleware chain.
      See verification.log for full test output.

This file is the source of truth for task completion. If it does not exist
when you finish, the orchestrator will treat your task as failed.

(For explore agents: the orchestrator writes output.yaml on your behalf.
Return your findings in the Task tool return message instead.

**Explore agent deviation reporting:**
If you examined files not listed in the plan, include in your return:
```
deviations:
  - type: files_examined_not_in_plan
    description: "Examined src/utils/helpers.ts for context"
    severity: minor
    justification: "Found reference to helper functions that affect the routing logic"
```
Or `deviations: []` if you only examined planned files.)
```

## Phase 3: Plan Validation

Before executing any tasks, validate the plan for correctness and completeness. Do not blindly trust that a plan is ready to run.

### Empty manifest

If the manifest contains zero tasks, the run transitions directly to `completed` after validation. No execution, verification, or commit phases are needed. Report this to the user.

### Structural integrity

- The DAG is acyclic (no circular dependencies)
- All `depends-on` and `receives` references point to task IDs that exist in the manifest
- All `receives` entries are a subset of the task's `depends-on`
- Every task in the manifest has a corresponding directory with a `plan.md` file
- No orphaned task directories (directories without a manifest entry)
- Task directory names follow the naming convention

### Completeness

- Each task's `plan.md` has a clear, specific objective (not vague or underspecified)
- Tasks that modify files specify which files they will touch
- The union of all tasks' work plausibly covers the stated goal -- look for obvious gaps
- **Requirements coverage**: Every requirement from the spec (or user request) is addressed by at least one task
  - Check the "Requirements Covered" section in each task's `plan.md`
  - Verify no requirements are orphaned (no task covers them)
  - Verify no tasks are orphaned (don't cover any requirement)
- Verification commands in the manifest actually exist in the project

### Coherence

- No two independent tasks (no dependency relationship) plan to modify the same file. If they do, they must be sequenced via `depends-on` or merged into one task.
- Tasks that `receive` upstream context actually need it -- the upstream task's described output is relevant to the downstream task's objective
- The DAG ordering makes sense -- tasks do not depend on unrelated tasks purely for serialization
- Constraints across tasks do not contradict each other

### Parallelization Constraints

**Tasks that share mutable state cannot run in parallel.** When tasks operate within the same worktree/repository and affect shared resources, they MUST be serialized via `depends-on` edges, even if they don't directly depend on each other's outputs. Common examples:

- **Mutating git operations**: Tasks that commit (`git commit`), checkout branches (`git checkout`), modify the git index (`git add`, `git rm`), or change repository state must be serialized. Read-only operations (`git status`, `git log`, `git diff`) are safe to run in parallel. Two simultaneous `git commit` operations will corrupt the repository state.
- **Build systems**: Tasks running `make`, `npm run build`, or similar must be serialized if they write to shared build artifacts, caches, or output directories.
- **File writes**: Tasks writing to the same file (caught in Coherence check) or files with shared dependencies (e.g., two tasks appending to the same log file).
- **Test databases**: Tasks that reset or mutate shared test databases must be serialized.

**The rule**: If two tasks would interfere with each other when run simultaneously in the same repository, add a `depends-on` edge to enforce ordering. Do not rely on timing or assume "it will probably be fine."

### Feasibility

- Referenced files actually exist in the codebase, or are explicitly created by this task, or are explicitly created by an upstream dependency (as stated in that task's plan)
- The agent type assigned to each task is appropriate -- `explore` agents cannot write files, so implementation tasks must not use them
- The scope of each task is reasonable for a single subagent

### Validation outcome

| Outcome | Action |
|---|---|
| **Valid** | Proceed to execution. Report validation results to user. |
| **Warnings** | Present warnings (e.g., "tasks A and B both modify `config.ts` but are not sequenced"). Ask whether to proceed or adjust. |
| **Blocking issues** | Present issues (e.g., "circular dependency between X and Y", "task Z references non-existent file"). Do not proceed until resolved. |

For simple issues (like a missing dependency edge between tasks that touch the same file), fix the plan and propose the fix to the user. For ambiguous issues, ask.

On resume, re-validate the remaining (non-completed) portion of the DAG before continuing.

## Phase 4: Execution Engine

Update `dispatch.yaml` status to `in-progress` when entering this phase for the first time.

Before dispatching any tasks, verify the git working tree is clean (no uncommitted changes outside the `dispatch/` directory). If there are uncommitted changes, warn the user and ask how to proceed -- Phase 6 commits could otherwise include unrelated modifications.

### Step 1: Resolve ready set

Read `dispatch.yaml`. Find all tasks where:

- Status is `pending`
- All tasks in `depends-on` are satisfied

A dependency is satisfied when:
- Its status is `completed`, OR
- Its status is `fixing` AND the current task's `fixes` field contains the dependency's task ID

Regular downstream tasks require all dependencies to be `completed`. Only fix tasks are allowed to proceed when a dependency is `fixing`, and only for the specific original task(s) listed in their `fixes` field. This prevents regular downstream tasks from being dispatched against code that was just rejected by critique or verification.

These are the "ready set."

If the ready set is empty and non-completed tasks remain, there is a deadlock (circular dependency) or all remaining tasks depend on failed tasks. Report this to the user and ask how to proceed.

### Step 2: Batch and fan out

Take up to `max-parallel` tasks from the ready set, accounting for any currently dispatched tasks or running critique agents (total concurrent subagents must not exceed `max-parallel`).

**Critical**: Ensure all tasks in the batch can safely run in parallel. Tasks that share mutable state (git operations, build commands, writing the same files) must not be in the same batch - they should have been serialized via `depends-on` edges during planning (see "Parallelization Constraints" in Phase 3).

For each task:

1. Read the task's `plan.md`
2. If `receives` (or `depends-on` when `receives` is omitted) references upstream tasks, read their `output.yaml` files and inject the content into the subagent's prompt as upstream context. Do not mutate `plan.md` on disk -- the upstream context is only included in the prompt sent to the subagent.
3. Tell the subagent the repository root path and the relative path to its task directory. Example: "Repository root: /home/user/project. Your task directory: dispatch/migrate-routes/1a-extract_auth_module/. Write output.yaml to your task directory."
4. Update the task's status to `dispatched` in `dispatch.yaml`
5. Dispatch via the Task tool using the `agent` type from the manifest

For `explore` agents (read-only): the orchestrator reads the Task tool return message as the task's output and writes `output.yaml` on the agent's behalf, since explore agents cannot write files.

**What the orchestrator writes for explore agents:**
- `status`: completed | failed
- `files-modified`: [] (explore agents are read-only)
- `notes`: Task tool return message
- `exports`: any structured data extracted from the return (empty object if none)
- `deviations`: extracted from the return message if present (empty list if none)

**Extracting deviations:**
- Parse the return message for a `deviations:` section
- If found: extract and include in output.yaml
- If not found: write `deviations: []`

If the explore agent's return indicates failure, set `status: failed` and populate `error`.

Dispatch all tasks in the batch as parallel Task tool calls in a single message.

### Step 3: Fan in

Collect results from all dispatched tasks:

1. **Check `output.yaml`** (source of truth):
   - Exists with `status: completed`: proceed to build verification (Step 3a)
   - Exists with `status: failed`: mark task `failed` in manifest
   - Does not exist: mark task `failed` (contract violation)

2. **Read the Task tool return message** (advisory):
   - Use for diagnostic context if the task failed
   - Sanity check against `output.yaml` -- if they conflict, log a warning and trust `output.yaml`

3. **Process deviations**:
   - Check `deviations` field exists in output.yaml (missing = task FAILED)
   - For each deviation in the list:
     * If severity >= configured threshold: escalate to user immediately
     * If severity < threshold: Karen evaluates and decides:
       - Accept deviation and continue
       - Create fix task to address deviation
       - Consult greybeard if technical judgment needed
   - Karen has authority to handle minor/moderate deviations autonomously
   - Major deviations or uncertainty → escalate to user

### Step 3a: Check Task Self-Report

Tasks self-report their verification status. Check that the report is complete:

1. **Check output.yaml exists** and has required fields including `verification-summary`
2. **Check evidence files exist** in the task directory:
   - For automated: `verification.log`, potentially `test-results.json`
   - For manual: `manual-evidence.log`
   - For review: `verification.log`
3. **Check evidence is not empty** - files have content, not just placeholder text

If evidence is missing or empty:
- Mark task as `fixing`
- Create fix task to add verification evidence
- Skip critique

If evidence exists and looks reasonable:
- Proceed to Step 3b (Critique) if critique is enabled for this task
- Mark task `completed` if critique is disabled

### Step 3b: Critique (if enabled)

If critique is enabled for this task (via global config or per-task override), do not mark the task `completed` yet. Instead:

1. Spawn a critique agent for the task (using the `agent` type from the critique config, defaulting to `critique`). The critique agent receives:
   - The original `plan.md` (what was asked for)
   - The `output.yaml` (what the doer claims it did)
   - Access to read the actual file changes in the codebase
   - **Access to read evidence files** in the task directory (verification.log, test-results.json, manual-evidence.log)
   - The repository root and task directory paths
   - The custom `critique.prompt` if provided, otherwise the default critique prompt (below)
2. The critique agent writes `critique.yaml` to the task directory
3. Based on the verdict:
   - `accepted`: mark the task `completed`
   - `needs-work`: mark the task `fixing`, create a fix task (see Fix Task Creation), and add it to the manifest

If the critique agent fails to produce a valid `critique.yaml` (crashes, times out, or writes a malformed file), treat the critique as `accepted` and mark the task `completed`. Log a warning that critique was skipped due to infrastructure failure. The doer's work should not be blocked by a critique agent malfunction.

Critique agents count toward `max-parallel`.

If critique is not enabled for the task, mark it `completed` immediately.

For `explore` agents, critique works differently since there are no file changes to review. Instead, the critique agent verifies the explore agent's findings against the actual codebase:

1. Spawn a critique agent (using the `agent` type from the critique config, defaulting to `critique`) that receives:
   - The original `plan.md` (what was asked)
   - The `output.yaml` (the explore agent's findings, written by the orchestrator)
   - The repository root and task directory paths
   - The explore critique prompt (below) or a custom `critique.prompt` if provided
2. The critique agent independently checks the codebase to verify key claims -- file existence, pattern identification, function signatures, module structure, etc.
3. The critique agent writes `critique.yaml` to the task directory
4. Based on the verdict:
   - `accepted`: mark the task `completed`
   - `needs-work`: mark the task `fixing`, create a fix task. The fix task is an `explore` agent that re-investigates, receiving the critique issues as context. The orchestrator writes a new `output.yaml` on the fix task's behalf as usual for explore agents.

#### critique.yaml format

```yaml
verdict: accepted | needs-work
issues:
  - severity: blocking | warning | nit
    file: src/routes/users/create.ts
    description: "Error handler swallows the original error message"
notes: |
  Overall approach is sound.
  Two issues need addressing.
```

Only `blocking` issues result in a `needs-work` verdict. Warnings and nits are recorded but do not block completion.

The critique agent uses the `critique` agent type. The agent must be able to write `critique.yaml` to the task directory. The critique prompt instructs it only to review and report. If a critique agent modifies source files, treat this as a bug and discard those changes. To detect this, the orchestrator should snapshot modified files (via `git status` or equivalent) before dispatching the critique agent and revert any newly modified files outside the task directory afterward.

#### Default critique prompt

When no custom `critique.prompt` is provided:

```
You are reviewing the work of another agent. You have:

1. The original plan (plan.md) describing what was asked
2. The agent's self-reported output (output.yaml)
3. The actual file changes in the codebase
4. The verification evidence files in the task directory

Evaluate whether:
- The objective in plan.md was met
- The files-modified list in output.yaml is accurate
- The code changes are correct and complete
- No unintended side effects were introduced
- Constraints from the plan were respected
- **Verification was actually performed:**
  - For automated: Check verification.log shows tests/builds were run
  - For manual: Check manual-evidence.log demonstrates the task works
  - For review: Check verification.log shows compile/lint/tests were run
- **Verification evidence matches the summary** in output.yaml

Important: Issues that must be marked as blocking:
- Objective not met
- Verification not performed (missing or fake evidence)
- Verification evidence contradicts the summary
- Code changes don't match what was claimed
- Build/test/lint failures in evidence

Write critique.yaml to your task directory with your verdict
(accepted | needs-work) and a list of specific issues with severity
(blocking | warning | nit).
Only blocking issues should result in a needs-work verdict.
```

#### Default explore critique prompt

When critiquing an `explore` agent and no custom `critique.prompt` is provided:

```
You are verifying the research findings of another agent. You have:

1. The original plan (plan.md) describing what was asked
2. The agent's findings (output.yaml)

Your job is to independently verify key claims against the actual
codebase. Do NOT take the findings at face value.

Check whether:
- Referenced files and directories actually exist
- Function signatures, type definitions, and exports match what was reported
- Patterns described (e.g., "all routes use X middleware") are accurate
- Important files or patterns were not missed
- The conclusions follow from the evidence in the codebase
- **Files examined match plan:** Check deviations field in output.yaml. If agent examined files not in plan, verify the deviation is reasonable for research purposes.

Write critique.yaml to your task directory with your verdict
(accepted | needs-work) and a list of specific issues with severity
(blocking | warning | nit).
Only blocking issues should result in a needs-work verdict.
A finding that downstream tasks will rely on being wrong is blocking.
```

### Step 3c: Fix task resolution

After processing all tasks in the batch (including build verification and critique), check every newly completed task: if it has a `fixes` field, transition each referenced original task's status from `fixing` to `completed` in the manifest. When `fixes` is a list, transition all referenced originals. Only apply this transition when the original task's current status is `fixing`.

This is what unblocks downstream tasks. Downstream tasks `depends-on` the original task ID, not the fix task. When the original transitions to `completed`, downstream tasks enter the ready set in the next iteration of Step 1.

Note: if a fix task itself is rejected by critique, the fix task transitions to `fixing` and a new fix task is created (e.g., `1a-fix2`) with `fixes` pointing to the same original task (not to the previous fix task). The rejected intermediate fix task (`1a-fix1`) stays at `fixing` permanently -- this is expected and cosmetic, since nothing references it in a `fixes` field. Only the original task's status matters for unblocking downstream dependents.

### Step 4: Unexpected modification detection (Safety Net)

**Purpose:** Catch deviations that tasks failed to report in Step 3. This is a safety net for dishonesty or oversight.

If `unexpected-modifications` is set to `accept` in the manifest, skip this step.

After each batch completes, compare each task's actual `files-modified` (from `output.yaml`) against the files listed in its `plan.md` under "Files to Modify". This comparison is best-effort since `plan.md` uses free-form markdown; extract file paths from backtick-quoted segments in the list items. For fix tasks (whose `plan.md` may not have a formal "Files to Modify" section), compare against the original task's planned files and any files mentioned in the error output that triggered the fix:

1. **Check if deviation was reported:**
   - If files were modified outside plan AND deviation was reported in Step 3: Already handled, no action needed
   - If files were modified outside plan AND deviation NOT reported: Task FAILED for dishonesty

2. **Handle unreported modifications:**
   - Mark task as `failed` (not `fixing` - this is a trust violation)
   - Error message: "Task modified [files] without reporting deviation in output.yaml. Plan specified: [planned files]"
   - Do NOT create fix task - require task to be re-run with proper deviation reporting

3. **Multiple tasks modifying same file:**
   - If both tasks reported the deviation in Step 3: Acceptable coordination issue
   - If either task didn't report: Both tasks FAILED

**Why this is strict:** Tasks are required to self-report deviations. Not reporting is a contract violation, not an implementation error. This step ensures accountability.

**Note:** The user may be asked about the same pattern across multiple batches. If this becomes disruptive, the user can set `unexpected-modifications: accept` in the manifest (not recommended for production use).

### Step 5: Update and repeat

Update `dispatch.yaml` with all status changes. Return to Step 1.

Continue until all tasks have status `completed` or `failed`.

If all tasks have status `failed` (or a mix of `failed` and `fixing` with no pending fix tasks), skip Phases 5/6/7. Set the run status to `failed` and report a failure summary to the user: which tasks failed, what errors occurred, and what fix attempts were made.

## Escalation

The orchestrator tracks escalation by counting the fix depth for a given original task: the number of tasks in the manifest whose `fixes` field references that original task's ID. For example, if `1a-fix1` and `1a-fix2` both have `fixes: 1a-extract_auth_module`, the fix depth is 2. When a fix task references multiple originals via `fixes`, escalation uses the maximum fix depth across all referenced originals.

- Depth 1-2: Fix silently, report progress in manifest
- Depth 3: Notify user that a task is proving difficult, explain what is failing
- Depth 4+: Ask user for guidance before continuing. Present the error, what has been tried, and options

This escalation policy applies to both verification failures and critique rejections. The counter is shared -- if critique produced fix1 and fix2, and then verification produces fix3, the user is notified at fix3 (depth 3) regardless of which mechanism created the fixes.

There is no hard retry limit. Escalation ensures the user is involved when things are not converging. Note: when a fix task is itself rejected by critique and a new fix task is created, both contribute to the fix depth. This is conservative -- it may over-count relative to the number of distinct fix attempts, but ensures the user is involved sooner rather than later.

## Phase 5: Full Build Verification

After all tasks are completed (and critiques passed, if enabled):

If no tasks modified any files (`files-modified` is empty across all completed tasks at the time Phase 5 begins), skip to Phase 7.

### Run Full Build

Consult project build documentation to determine the complete verification suite. A full build typically includes:
- **Generate** (if applicable): Code generation step
- **Lint**: Project-wide linting
- **Build**: Full compilation/build
- **Test**: Complete test suite

Run the commands identified from project documentation (README.md, CONTRIBUTING.md, Makefile, package.json scripts, etc.)

Save output to `dispatch/<run-name>/final-build.log`

### Compare to Baseline

Compare final build results to `baseline-build.log` (captured at dispatch start):

1. **If final build passes and baseline passed**: Success, proceed to Phase 6
2. **If final build fails and baseline failed with same errors**: No regression introduced, proceed to Phase 6
3. **If final build has NEW failures not in baseline**: Regression introduced, enter fix loop

### Attribution and Fix Loop

When regressions are detected:

1. **Attribute failures to tasks**: Launch a subagent (general or explore agent) to analyze:
   - Which files have errors
   - Which tasks modified those files
   - Map errors to responsible task(s)
   - Save attribution to `dispatch/<run-name>/failure-attribution.md`

2. **Create fix tasks**: For each task with attributed failures
   - Mark original task as `fixing`
   - Create fix task with error details and attribution

3. **Re-run Phase 4**: Execute fix tasks through normal dispatch flow

4. **Re-verify**: Run full build again, compare to baseline

5. **Repeat** until final build matches baseline (no new failures)

### Fix Loop Details

The fix loop handles regressions detected in Phase 5:

1. **Attribution subagent analyzes**:
   - Reads `baseline-build.log` and `final-build.log`
   - Identifies new failures (not present in baseline)
   - Maps failures to files
   - Maps files to tasks that modified them
   - Writes `failure-attribution.md` with analysis

2. **Mark originals**: For each task with attributed failures, transition status from `completed` to `fixing`

3. **Create fix tasks**: See Fix Task Creation section

4. **Execute fixes**: Re-enter Phase 4 (Steps 1-5)
   - Fix tasks execute normally
   - Include critique if enabled
   - Step 3c transitions originals to `completed` when fixes succeed

5. **Re-verify**: Run full build again, compare to baseline
   - If matches baseline: exit loop, proceed to Phase 6
   - If new failures remain: return to step 1

6. **Cascade check**: If a fix changed interfaces affecting downstream tasks:
   - Attribution subagent detects cross-task failures
   - Attributes to both the fix task and affected downstream tasks
   - Create fix tasks for all involved parties

7. **Repeat** until verification passes or escalation triggers user intervention

## Fix Task Creation

This section applies to fix tasks created by both critique rejection (Step 3a) and verification failure (Phase 5 fix loop).

### Naming

Fix tasks use the format `<original-level><original-sequence>-fix<n>-<description>`:
- `1a-fix1-resolve_critique_issues`
- `1a-fix2-handle_error_edge_case`

The orchestrator determines `<n>` by counting existing fix tasks for the same original in the manifest and adding one. When a fix task references multiple originals (cross-task fix), the prefix uses the first original listed in `fixes` (alphabetically by task ID), and `<n>` is determined by counting all existing fix tasks whose `fixes` field contains any of the referenced originals, then adding one.

### Manifest entry

```yaml
# Single-task fix
- id: 1a-fix1-resolve_critique_issues
  agent: general
  depends-on: [1a-extract_auth_module]
  receives: []
  fixes: 1a-extract_auth_module
  status: pending
  commit-group: auth         # inherited from the task being fixed

# Cross-task fix (verification failure involving multiple tasks)
- id: 1a-fix2-align_auth_interface
  agent: general
  depends-on: [1a-extract_auth_module, 2a-integrate_modules]
  receives: []
  fixes: [1a-extract_auth_module, 2a-integrate_modules]
  status: pending
```

Key fields:
- `fixes`: references the original planned task ID or a list of IDs (not previous fix tasks). When this fix task completes, the execution engine (Step 3b) transitions each referenced original task from `fixing` to `completed`. Use a list when a single fix addresses a cross-task interaction (e.g., mismatched interfaces between two tasks).
- `depends-on`: includes the original task(s) being fixed. Since the originals have status `fixing` and this task has a `fixes` field, Step 1 treats those dependencies as satisfied. Do NOT include previous fix tasks (e.g., `1a-fix1`) in `depends-on` -- they may have status `failed`, which would deadlock the new fix. Ordering between fix attempts is implicit: the orchestrator creates fix tasks sequentially, so `1a-fix2` is only created after `1a-fix1` has finished and failed. When a verification failure involves multiple tasks (cross-task interaction), the fix task's `depends-on` should include all attributed original tasks, and the plan should reference relevant output from all of them.
- `receives: []`: fix tasks MUST set `receives: []` explicitly. Omitting this field would cause `receives` to default to `depends-on`, which would inject the original task's `output.yaml` -- a document that claims `status: completed` despite the task having been rejected. Instead, all context is included directly in the fix task's `plan.md` (see Plan content below).
- `commit-group`: inherited from the original task's `commit-group`. If the original had no group, the fix task has no group. When `fixes` is a list and the referenced originals have different `commit-group` values, the fix task has no group (it gets its own commit under the `grouped` strategy). If all referenced originals share the same group, the fix task inherits it.

### Plan content

The fix task's `plan.md` includes:
- The error output or critique issues that triggered the fix
- The original task's objective (for context)
- The original task's upstream context (from its `receives` chain -- the orchestrator looks up what the original task received and includes the same upstream context in the fix task's plan)
- Specific instructions on what to fix

### Directory

Each fix task gets its own directory as a scratchpad, following the naming convention: `1a-fix1-resolve_critique_issues/plan.md`.

## Phase 6: Commits

After all verification passes, orchestrate commits for the changes.

### Commit strategies

| Strategy | Behavior |
|---|---|
| `per-task` | One commit per completed task, in topological order. See note below on shared files. |
| `single` | All changes in one commit. Message summarizes the overall goal. |
| `grouped` | Tasks sharing a `commit-group` value go into one commit. Tasks without a group get their own. Fix tasks inherit `commit-group` from the task they fix. |

**`per-task` and shared files**: Since all tasks run before commits are created, `git add` stages the file's current (final) state, not intermediate states. When dependent tasks modify the same file, the file is attributed to the **last task in topological order** that modified it. Earlier tasks' commits will not include that file. This means `per-task` is a logical attribution of the commit message, not a reconstruction of intermediate file states. Reverting an earlier task's commit will not revert all changes that task made if some of its files were attributed to a later commit. Fix tasks are ordered after all originals they fix (their `depends-on` edges determine topological position naturally) and will typically absorb the files they fixed, but the general file attribution rule (last in topological order) still applies. If a task's commit would contain zero files (all attributed to later tasks), skip that commit. If a run involved fix tasks and clean commit history matters, prefer the `single` or `grouped` strategy instead.

### Approval modes

| Mode | Behavior |
|---|---|
| `ask-once` (default) | Present the full commit plan (all commits with files and messages). User approves or adjusts, then all commits execute. |
| `ask-each` | Present each commit individually for approval. User can approve, edit the message, or skip. |
| `auto` | Commits execute without user interaction. Verification already passed. |

### Commit phase steps

1. Read commit strategy and approval mode from manifest
2. Sort tasks by topological order, breaking ties alphabetically by directory name (or group by `commit-group`)
3. Build the commit plan:
   - `per-task`: each task becomes one commit using its `files-modified`
   - `grouped`: merge `files-modified` across tasks in the same group
   - `single`: merge all `files-modified` into one commit
4. Generate commit messages:
   - `message-source: objective`: derive from the task's plan.md objective
   - `message-source: notes`: derive from the task's output.yaml notes
   - Apply `prefix` if configured
   - For `grouped` commits, synthesize the objectives/notes of all tasks in the group into a single message. For `single` commits, derive the message from the `goal` field in the manifest, using `message-source` as guidance for tone.
5. Apply approval mode:
   - `ask-once`: present full commit plan, wait for approval
   - `ask-each`: present each commit, wait for approval
   - `auto`: proceed immediately
6. Execute commits: `git add <files>` then `git commit` for each unit
7. Record commit SHAs in `dispatch.yaml` under `results.commits` (list of `{sha, message, files, tasks}`)

## Phase 7: Completion

When all tasks are completed, verification passes, and commits are done:

1. Update `dispatch.yaml` status to `completed`
2. Report a summary:
   - Total tasks executed
   - Fix rounds needed (if any)
   - Critique rounds (if any)
   - Files modified across all tasks
   - Commits created
   - Verification results
3. Ask the user whether to keep or clean up the `dispatch/` directory

## Resumability

When `$ARGUMENTS` points to an existing `dispatch.yaml`:

If the run status is `completed`, report the previous run's results and exit.

If the run status is `failed`, present the failure summary and ask the user whether to retry failed tasks (reset them to `pending` and set run status to `in-progress`) or abort.

If the run status is `pending`, re-run Phase 3 validation and ask the user for approval before proceeding to Phase 4. The orchestrator was likely interrupted between plan approval and execution start.

If the run status is `in-progress`:

1. Read the manifest
2. Re-validate the remaining (non-completed) portion of the DAG (Phase 3)
3. Resume based on task status:
   - `completed`: skip
   - `dispatched`: check whether `output.yaml` exists in the task directory. If it does, the subagent likely completed before the orchestrator was interrupted -- process the existing `output.yaml` through Steps 3/3a/3b instead of re-dispatching. If `output.yaml` does not exist, reset to `pending`.
   - `failed`: present to user, ask whether to retry or skip
   - `fixing`: check the associated fix task(s) (found by scanning for tasks whose `fixes` field references this task):
     - If the fix task is `dispatched`: apply the same `output.yaml` check as regular dispatched tasks above
     - If the fix task is `completed` but the parent is still `fixing`: transition the parent to `completed`
     - If the fix task is `failed`: present to user, ask whether to retry or skip
     - If the fix task is `pending`: leave both as-is; the fix task will be picked up by the ready set in Phase 4
     - If no fix task exists: treat the original as `failed`, present to user
   - `pending`: proceed normally
4. Resume from Phase 4, Step 1

If `results.commits` is non-empty but run status is `in-progress`, the orchestrator was likely interrupted during Phase 6. Check which commits were already created (verify SHAs exist in the git log), skip those, and continue committing the remaining tasks. If all planned commits already exist, proceed directly to Phase 7.

The orchestrator does not detect external changes to the working tree between interruption and resume. If files were manually modified, Phase 5 verification will catch build/test/lint failures, but unrelated manual changes that don't break verification may be silently included in Phase 6 commits. For a clean resume, the user should ensure the working tree has no unrelated modifications.

## Source of Truth Contract

The orchestrator and subagents agree on the following:

| Artifact | Role | Who writes | Who reads |
|---|---|---|---|
| `plan.md` | Task instructions | Orchestrator | Subagent |
| `output.yaml` | Task results (source of truth) | Subagent (Orchestrator for `explore` agents) | Orchestrator |
| `critique.yaml` | Review verdict | Critique agent | Orchestrator |
| Task tool return | Diagnostic supplement (advisory) | Subagent | Orchestrator (diagnostics only) |
| `dispatch.yaml` | Run state and DAG | Orchestrator | Orchestrator |
| Other files in task dir | Scratch space | Subagent | Nobody (unless referenced in exports) |

**`output.yaml` is the source of truth for task completion.** The Task tool return message is advisory -- used for diagnostics when things fail and as a sanity check against `output.yaml`. If they conflict, `output.yaml` wins.

If `output.yaml` does not exist when a subagent finishes, the task is treated as failed regardless of what the Task tool return says.

Exception: for `explore` agents, the orchestrator writes `output.yaml` on the agent's behalf using the Task tool return message, since explore agents cannot write files.

## Guiding Principles

From the philosophy skill:

- **Pragmatic over idealistic** -- Do not over-plan. A single task with no dependencies is a valid dispatch.
- **Simple is usually harder than easy** -- The DAG should reflect real dependencies, not imagined ones. Do not serialize tasks that can run in parallel.
- **Do no harm** -- Verify before declaring victory. Verification commands exist for a reason. The style skill says "do not work around a failing build" -- dispatch's fix loop is not a workaround; it is an active attempt to resolve the failure. If the fix loop cannot converge, escalation ensures the user is involved.
- **Constraint ownership** -- Each subagent owns its task directory. The orchestrator owns the manifest. Do not cross boundaries.

### Parallelization Safety

**Not all tasks can run in parallel.** Tasks sharing mutable state within the same repository must be serialized:

- **Mutating git operations** (commit, checkout, stash, add, rm) modify global repository state and must be serialized. Read-only git operations (status, log, diff) are safe to run in parallel.
- Build systems write to shared output directories and caches
- File operations targeting the same paths create race conditions

When in doubt, add a `depends-on` edge. Parallelism is an optimization; correctness is mandatory. A conservative DAG that serializes potentially conflicting tasks is better than a corrupted repository state.
