---
name: dispatch
argument-hint: "[<name> | dispatch/<name>/ | dispatch/<name>/dispatch.yaml | <spec-file> ]"
description: Orchestrate parallel subagent task runs. Smart input resolution - provide a name, directory, yaml file, or spec file. No argument runs the latest dispatch.
---

# Dispatch

Orchestrate parallel subagent task runs across a dependency graph. Fan out work to subagents, fan in results, validate, critique, verify, fix failures, commit, and repeat until done.

## Terminology

**Important:** Dispatch is a **skill** you load with `skill(name="dispatch")`, not a command to run.

- **"Run"** or **"the run"** = One complete instance of dispatch orchestrating tasks
- **"Loading dispatch"** = Starting the skill (not "invoking" or "executing")
- **"Running tasks"** = When subagents perform work

Avoid "execute dispatch" - use "run dispatch" or "load dispatch skill" instead.

## Input Resolution

Dispatch figures out what to run based on the argument:

**No argument** → Run latest dispatch (newest `dispatch/<name>/` directory by creation time)

**Just a name** (e.g., "auth-fix") → Look for `dispatch/<name>/dispatch.yaml`

**Directory** (e.g., `dispatch/auth-fix/`) → Look for `dispatch.yaml` inside

**File** → Check filename:
- Ends with `dispatch.yaml` → Run it
- Anything else → Treat as spec, plan first, then run

**On checkpoint decline (when planning from spec):**
- Planning artifacts remain in `dispatch/<run-name>/` for review
- Can re-run later with: `dispatch dispatch/<run-name>/dispatch.yaml`

## Phase 1: Planning

This phase runs when dispatch is loaded with a spec.md file as input. The spec should be complete and approved before dispatch is called.

**Prerequisites:**
- Spec file exists with complete requirements
- User has confirmed the spec is ready for implementation

**Input:** The spec file at the provided path

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

Present this verification to the user BEFORE presenting the full dispatch plan. This ensures quality requirements are addressed proactively, not discovered during the run.

**Why this matters:**

Late failure detection is expensive. If 15 tasks complete and Phase 5 verification fails, you must debug all 15 tasks to find the culprit. Grouped commits hide boundaries, making bisection impossible.

Early failure detection is cheap. If task 3a writes tests and immediately runs them, you know task 3a is broken and fix it in isolation.

The quality gates prevent the expensive scenario.

**After Phase 2 completes:** Proceed to the Presentation Checkpoint (see Checkpoint section below)

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

Task directories are named so a human scanning the directory can immediately understand run order and purpose. The format is:

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

Each task directory is the subagent's scratchpad. It reads its plan from there, does its work, and writes its results there. After running, a task directory looks like:

```
1a-extract_auth_module/
  plan.md              # input: instructions for the subagent
  output.yaml          # output: structured results (source of truth)
  ...                  # scratch: any other files the subagent creates
```

Gate critique verdicts are written at the run level, not the task level:

```
dispatch/<run-name>/
  dispatch.yaml
  level1-gate-critique.yaml         # gate verdict for level 1 planned tasks
  level2-gate-critique.yaml         # gate verdict for level 2 planned tasks
  level1-fix-round1-gate-critique.yaml  # gate for first round of fix tasks at level 1
  level1-fix-round2-gate-critique.yaml  # gate for second round (if round 1 fix tasks fail)
  1a-extract_auth_module/
    plan.md
    output.yaml
    ...
```

A "fix round" for a level is the set of fix tasks that become ready at the same time. Round 1 is the first batch of fix tasks for that level; round 2 is any fix tasks spawned because round 1 fix tasks were themselves rejected; and so on. The round number equals 1 plus the count of existing `level<N>-fix-round*-gate-critique.yaml` files in the dispatch run directory for that level. On resume, stale or malformed gate files from a previously aborted run may inflate the counter by one, causing a gap in the sequence (e.g., round 3 instead of round 2). This is cosmetically imperfect but has no behavioral consequence — gate file names are informational artifacts only and are not used by the execution engine for logic.

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
  agent: critique            # defaults to critique; must be an agent that can write gate-critique.yaml

commits:
  strategy: per-task         # per-task | single | grouped
  approval: ask-once         # ask-once | ask-each | auto
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
      validate-fix:          # optional: enable mutation testing validation; requires per-task commit strategy; most reliable when no downstream task re-touches the same files
        enabled: false       # opt-in; must be explicitly enabled
        test-command: ""     # command to run test(s) for validation
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
| `status` | yes | Current run status |
| `critique` | no | Per-task critique config; overrides the global `critique` setting |
| `commit-group` | no | Group name for the `grouped` commit strategy. Fix tasks inherit this from the task they fix. |

### Task status values

| Status | Meaning |
|---|---|
| `pending` | Not yet ready to dispatch (dependencies incomplete) |
| `dispatched` | Currently being run by a subagent |
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

## Subagent Responsibility: Read and Understand the Plan

**Before starting work, you MUST:**

1. **Read your plan.md file** from your task directory (the orchestrator will tell you the path)
2. **Confirm you understand:**
   - The objective and what success looks like
   - All requirements covered by this task
   - Which files to modify and what changes to make
   - All constraints (what NOT to do)
   - How to verify the task is complete
3. **If the plan is unclear, contradictory, or missing critical information:**
   - STOP immediately
   - Do NOT attempt to guess or proceed with partial understanding
   - Report failure in output.yaml with a clear description of what is unclear or missing

**Why this matters:** The orchestrator provides a summary in the dispatch prompt, but the plan.md file is the authoritative source of truth. The summary may not include all constraints, edge cases, or detailed requirements. Relying solely on the summary can lead to incorrect implementations and wasted effort.

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

### Mandatory boilerplate sections

The following sections from the template above are **protocol sections** — they do not vary per task. You MUST copy them verbatim into every `plan.md` you create. Do not paraphrase, abbreviate, or omit them:

- **Deviation Reporting** (the full section including the yaml block, severity levels, and consequence warning)
- **Subagent Responsibility: Read and Understand the Plan** (the full section)
- **Output Contract** (the full section including the example yaml)

These sections define the communication protocol between subagents and the orchestrator. If they are missing or altered, subagents will produce output that violates the contract (e.g., writing `status: success` instead of `status: completed`), causing silent failures or incorrect orchestration behavior.

## Phase 2.5: Plan Critique

After creating the dispatch.yaml and all plan.md files, run a quality critique on the plan itself before presenting to the user. This catches planning gaps, missing requirements, and structural issues before tasks start.

**When to run:** Only on initial dispatch creation (not when resuming). Runs after Phase 2 completes and before Phase 3 validation.

**Process:**

1. **Spawn critique agent** with:
   - The dispatch.yaml manifest
   - All task plan.md files
   - The original spec file (if planning from spec)

2. **Critique scope:**
   - **Manifest level:** All spec requirements covered? DAG valid? Agent types appropriate? Dependencies correct?
   - **Task level:** Objectives clear? Files identified? Constraints realistic? Verification appropriate?
   - **Cross-task level:** No conflicting constraints? Shared resources properly sequenced?

3. **Critique output:** Writes `plan-critique.yaml` to the dispatch directory:

```yaml
verdict: accepted | needs-work
iteration: 1
timestamp: YYYY-MM-DDTHH:MM:SS
issues:
  - severity: blocking | warning | nit
    type: spec | plan  # NEW: distinguishes source of issue
    component: dispatch.yaml | 1a-task_name | specs/...
    description: "Specific issue found"
    suggestion: "How to fix it"
notes: "Overall assessment"
```

### Spec Issues vs Plan Issues

Critique must distinguish between:
- **Spec issues:** Requirements missing, conflicting goals, unclear scope, incomplete constraints
- **Plan issues:** Task breakdown problems, dependency errors, verification gaps

**Spec issue identification:**
When critique finds a spec issue, it tags it in plan-critique.yaml with `type: spec`.

### Spec Issue Resolution Workflow

When spec issues are found (any `type: spec` issues with severity blocking or warning):

1. **Pause planning** - Do not continue with plan fixes until spec is resolved
2. **Present to user:** "Critique found N spec issues that need resolution before planning can continue"
3. **User updates spec** - User edits `specs/<name>-spec.md` directly to address the issues
4. **Reset iteration counter** - Fresh start after spec fix
5. **Restart Phase 1** - Re-plan from updated spec

**Note:** Spec issues take precedence over plan issues. Fix spec first, then plan.

**Iteration policy (same as execution fix depth):**

- **Iterations 1-2:** Calling agent fixes issues (in-place edits to yaml/plan files), re-runs critique
- **Iteration 3:** Warn that plan is proving difficult to validate
- **Iteration 4+:** Ask user via question tool:
  - "Proceed anyway (accept plan with known issues)"
  - "Consult greybeard for architectural guidance"
  - "Abort dispatch and start over"
  - "Manual review needed"

**Spec Issue Exception:**
When spec issues are found and the user updates the spec:
- Reset iteration counter to 0
- Restart Phase 1 with updated spec
- This ensures clean planning from corrected requirements
- Spec fixes take precedence over iteration limits

**Escalation to greybeard:**

If calling agent cannot resolve critique issues, consult greybeard for architectural guidance on how to fix. This is a Task tool consultation, not automatic.

**Blocking behavior:**

- Plan critique is **mandatory** - no bypass option
- Must pass (verdict: accepted) before proceeding to Phase 3
- If critique agent crashes or produces invalid output: Block and ask user for decision

**Artifact persistence:**

Keep `plan-critique.yaml` as dispatch history for audit trail. Do not delete after successful validation.

## Phase 3: Plan Validation

Before running any tasks, validate the plan for correctness and completeness. Do not blindly trust that a plan is ready to run.

### Empty manifest

If the manifest contains zero tasks, the run transitions directly to `completed` after validation. No running, verification, or commit phases are needed. Report this to the user.

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

- No two tasks at the same level plan to modify the same file. If they do, they must be merged into one task. Adding a `depends-on` edge between same-level tasks to work around this is wrong — it just creates an artificial sequencing dependency where the work should have been one task.
- Tasks that `receive` upstream context actually need it -- the upstream task's described output is relevant to the downstream task's objective
- The DAG ordering makes sense -- tasks do not depend on unrelated tasks purely for serialization
- Constraints across tasks do not contradict each other

### Parallelization Constraints

**Tasks that share mutable state cannot run in parallel.** When tasks operate within the same worktree/repository and affect shared resources, they MUST be serialized via `depends-on` edges, even if they don't directly depend on each other's outputs. Common examples:

- **Mutating git operations**: Tasks that commit (`git commit`), checkout branches (`git checkout`), modify the git index (`git add`, `git rm`), or change repository state must be serialized. Read-only operations (`git status`, `git log`, `git diff`) are safe to run in parallel. Two simultaneous `git commit` operations will corrupt the repository state.
- **Build systems**: Tasks running `make`, `npm run build`, or similar must be serialized if they write to shared build artifacts, caches, or output directories.
- **File writes**: Tasks at the same level writing to the same file must be merged (caught as a blocking issue in the Coherence check). Tasks at different levels are already serialized by the DAG and do not conflict.
- **Test databases**: Tasks that reset or mutate shared test databases must be serialized.

**The rule**: If two tasks would interfere with each other when run simultaneously in the same repository, add a `depends-on` edge to enforce ordering. Do not rely on timing or assume "it will probably be fine."

### Feasibility

- Referenced files actually exist in the codebase, or are explicitly created by this task, or are explicitly created by an upstream dependency (as stated in that task's plan)
- The agent type assigned to each task is appropriate -- `explore` agents cannot write files, so implementation tasks must not use them
- The scope of each task is reasonable for a single subagent

### Validation outcome

| Outcome | Action |
|---|---|
| **Valid** | Proceed to running tasks. Report validation results to user. |
| **Warnings** | Present warnings (e.g., "tasks A and B depend on each other's outputs but the dependency edge is missing"). Ask whether to proceed or adjust. |
| **Blocking issues** | Present issues (e.g., "circular dependency between X and Y", "task Z references non-existent file", "tasks A and B are at the same level and both modify `config.ts`"). Do not proceed until resolved. |

For same-file same-level conflicts, the resolution is always to merge the two tasks — not to add a `depends-on` edge. Fix the plan and propose the merge to the user. For ambiguous issues, ask.

On resume, re-validate the remaining (non-completed) portion of the DAG before continuing.

## Checkpoint: Validation, Presentation, and Confirmation

This is the convergence point for both input types (existing dispatch.yaml or new spec.md). All flows pass through here before tasks start.

**Prerequisites:**
- Phase 2.5 (Plan Critique) completed with verdict `accepted`
- No unresolved `type: spec` issues (spec must be solid before planning)
- Plan reflects current spec state

For new dispatches created from spec, this ensures both the spec and plan have been reviewed for completeness and correctness.

### Step 1: Validate

Run Phase 3 validation on the dispatch:

- **Structural integrity**: DAG acyclicity, valid task references, required fields present
- **Completeness**: All tasks have clear objectives, files specified, requirements covered
- **Coherence**: No conflicting constraints, parallelization constraints respected
- **Feasibility**: Referenced files exist or will be created, appropriate agent types

**Validation outcomes:**
- **Valid**: Proceed to presentation
- **Warnings**: Present warnings, ask whether to proceed or adjust
- **Blocking issues**: Present issues, do not proceed until resolved

### Step 2: Present

Display the current state to the user before requesting confirmation.

**For Existing Dispatch (dispatch.yaml input):**
```
Dispatch: <run-name>
Status: <pending|in-progress>

Task Summary:
- Completed: X tasks
- Pending: Y tasks
- Failed: Z tasks
- In Progress: W tasks

Pending/Failed Tasks:
- 2a-<task>: Status: pending, Depends on: [1a, 1b]
- 3b-<task>: Status: failed, Error: <brief-error>

Ready to Resume: <true/false> (based on ready set availability)
```

**For New Dispatch (spec.md input):**
```
New Dispatch: <run-name>
Goal: <goal-description>
Spec: specs/<run-name>-spec.md

Quality Gate Verification:
✅ All code-producing tasks have build verification
✅ All test-writing tasks verify tests pass
✅ Commit strategy enables debugging (per-task)

Task DAG (X tasks total):
Level 1 (can run in parallel):
- 1a-<task>: <agent-type> - <description>
- 1b-<task>: <agent-type> - <description>

Level 2:
- 2a-<task>: depends on [1a] - <description>
- 2b-<task>: depends on [1b] - <description>

Level 3:
- 3a-<task>: depends on [2a, 2b] - <description>

Verification Commands:
- Build: <command>
- Test: <command>
- Lint: <command>

Critique Enabled: <yes/no> (X of Y tasks)
Commit Strategy: <per-task|grouped|single>

Dispatch files created at: dispatch/<run-name>/
```

### Step 3: Confirm

Use the question tool to ask the user whether to proceed. The question should offer "Proceed" and "Abort" as options.

**If user selects "Proceed":**
- Update `dispatch.yaml` status to `in-progress`
- Proceed to Phase 4 (Execution Engine)
- Execution proceeds linearly without returning to planning

**If user selects "Abort":**
- Exit cleanly without starting the run
- Dispatch files remain intact for manual review
- User can edit files directly, then re-run with: `dispatch dispatch/<run-name>/dispatch.yaml`

**If user wants modifications:**
The skill does NOT modify the plan interactively. The user must:
1. Edit the relevant files (`dispatch.yaml`, `plan.md` files)
2. Re-run with: `dispatch dispatch/<run-name>/dispatch.yaml`

This maintains the non-reentrant property - once a plan is confirmed and the run begins, it proceeds to completion.

### Why This Checkpoint Matters

1. **Visibility**: User sees exactly what will run before it starts
2. **Control**: User can abort or modify before any changes are made to the codebase
3. **Safety**: No accidental running of incomplete or incorrect plans
4. **Debugging**: User can review the plan offline before committing to the run

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
4. **Explicitly instruct the subagent to read its plan.md file.** Include text in the prompt such as: "You MUST read your complete plan.md file from your task directory before proceeding. The summary provided here is for context only. If any part of the plan is unclear, contradictory, or you cannot understand what is being asked, STOP and report failure."
5. Update the task's status to `dispatched` in `dispatch.yaml`
6. Dispatch via the Task tool using the `agent` type from the manifest

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
   - Exists with `status: completed`: proceed to self-report check (Step 3a)
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
- Hold the task in an internal `self-report-passed` state (in-memory only; not written to `dispatch.yaml`; cleared once Step 3b finishes for the level, whether the gate ran, was skipped, or failed — all tasks exit `self-report-passed` by transitioning to `completed` or `fixing` before Step 3b returns)
- Once all tasks in the current level have reached `self-report-passed`, `failed`, or `fixing`, proceed to Step 3b (which determines per-task critique eligibility and runs the gate if needed)

### Step 3b: Level Gate Critique (if enabled)

Once all tasks in the current level have passed Step 3a (or been marked `failed`/`fixing`), determine which tasks need critique. Per-task setting takes precedence over global. The global `critique.enabled` defaults to `true` when the `critique.enabled` key is absent — whether the entire `critique` block is omitted or the block exists but lacks the `enabled` key.

| Global `critique.enabled` | Per-task `critique.enabled` | Needs critique? |
|---|---|---|
| `true` (or key absent) | absent | yes |
| `true` (or key absent) | `true` | yes |
| `true` (or key absent) | `false` | no |
| `false` | absent | no |
| `false` | `true` | yes |
| `false` | `false` | no |

Tasks that do not need critique are marked `completed` immediately. Tasks that need critique form the gate input set. If the gate input set is empty, skip the gate entirely.

The gate input set is also restricted to tasks in `self-report-passed` state — tasks already marked `failed` or `fixing` (due to Step 3a evidence failures) are excluded. They are not re-evaluated by the gate.

If the gate input set is non-empty, spawn a single gate critique agent:

1. Spawn one critique agent. Agent type is selected as follows:
   - If all gate-input tasks share the same per-task `critique.agent` type → use that type
   - Otherwise (tasks have different per-task agent types, or no per-task override) → use the global `critique.agent`, defaulting to `critique`
   - Note: when tasks have heterogeneous per-task agent types, both custom preferences are discarded in favor of the global default. If this is unacceptable for a level, the tasks should be split into separate levels or given a shared per-task agent type.

   The critique agent receives:
   - For each task in the gate input set: its `plan.md`, its `output.yaml`, and the evidence files listed in its `verification-summary`
   - The repository root and task directory paths for each task
   - A prompt selected by applying these rules in order:
     1. If all gate-input tasks have a per-task `critique.prompt` and they are all identical → use that prompt
     2. Otherwise (some tasks have no per-task prompt, or prompts differ) → use the global `critique.prompt` if present
     3. Otherwise → if any gate-input task has `validate-fix.enabled: true`, use the validation gate prompt
     4. Otherwise → if all gate-input tasks are `explore` agents, use the explore gate prompt; else use the standard gate prompt
     - Same trade-off as agent type: heterogeneous or partial per-task prompts are discarded in favor of the global default
     - **Warning:** If a per-task or global `critique.prompt` is selected (rules 1 or 2) and any gate-input task has `validate-fix.enabled: true`, the custom prompt will be used as-is — mutation testing instructions are not automatically injected. Log a warning that validate-fix tasks are present but the custom prompt was used; mutation testing will not run unless the custom prompt includes those instructions.
2. The critique agent writes one `gate-critique.yaml` to `dispatch/<run-name>/`. For planned tasks the file is named `level<N>-gate-critique.yaml` (e.g., `level1-gate-critique.yaml`). For fix task gates it is named `level<N>-fix-round<R>-gate-critique.yaml` where R is the fix round number (see directory structure above).
3. Based on the verdict:
   - Tasks with no blocking issues → mark `completed`
   - Tasks referenced by blocking issues → mark `fixing`, create a fix task per affected task (see Fix Task Creation), add to manifest

Fix tasks for a level form their own mini-gate: when they complete Step 3a, a new gate critique runs scoped to just the fix task set (same per-task critique enablement rules apply).

**Single-task levels** follow the same path. When a level has only one task, the gate input set contains that one task and the gate runs as normal. This degenerates to the same behavior as the old per-task model, just with consistent mechanics.

If the critique agent fails to produce a valid `gate-critique.yaml` (crashes, times out, or writes a malformed file), treat the gate as `accepted` for all tasks in the gate input set and mark them `completed`. Log a warning that critique was skipped due to infrastructure failure.

Critique agents count toward `max-parallel`.

**Handling validation results:**

If `validate-fix` was enabled for a task:
1. Check whether `gate-critique.yaml` contains a `skipped-validation` entry where `task-id` matches the task. If present:
   - `type: structural`: log the reason and treat the task as if mutation testing passed. Do not block.
   - `type: anomaly`: log the reason as a **warning** to the operator and treat the task as if mutation testing passed. Do not block, but the operator should investigate why files span multiple commits or why files-modified is empty.
2. Otherwise, check that `gate-critique.yaml` contains a `validation` list entry where `task-id` matches the task. If the entry is absent (and no `skipped-validation` entry exists), treat this as a blocking issue — mutation testing was required but not performed (likely because a custom prompt was used that does not include mutation testing instructions; see the warning in the prompt selection rules above)
3. If the `validation` entry is present, check its `mutation-test` field: verify without-fix.status is "failed" and with-fix.status is "passed"
4. If without-fix.status is not "failed" or with-fix.status is not "passed", treat as a blocking issue (needs-work) for that task
5. Evidence files (mutation-test-*.log) remain in the task directory for reference

**Handling new test files:**

For each task being processed, find the entries in `gate-critique.yaml`'s `new-tests` list where `task-id` matches that task, then:
1. Add each matched `file` to that task's `files-modified` list in its `output.yaml`
2. They will be included when the task's commit is created

For `explore` agents (whether in a pure-explore level or mixed with other agent types), the gate critique verifies findings against the actual codebase rather than reviewing file changes. The gate agent independently checks the codebase to verify key claims from each explore task's output.yaml: file existence, pattern identification, function signatures, module structure, etc. For a `needs-work` verdict on an explore task, the fix task is an `explore` agent that re-investigates, receiving the critique issues as context. The orchestrator writes a new `output.yaml` on the fix task's behalf as usual for explore agents.

#### gate-critique.yaml format

```yaml
tasks: [1a-task, 1b-task]     # all task IDs in scope for this gate run
verdict: accepted | needs-work
issues:
  - task-id: 1b-task           # which task this issue belongs to
    severity: blocking | warning | nit
    file: src/routes/users/create.ts
    description: "Error handler swallows the original error message"
validation:                    # present only if validate-fix was enabled for a task
  - task-id: 1a-task
    mutation-test:
      fix-commit: abc123
      without-fix:
        status: failed
        evidence-file: mutation-test-without-fix.log
      with-fix:
        status: passed
        evidence-file: mutation-test-with-fix.log
skipped-validation:            # present only if validate-fix was enabled but mutation testing was skipped
  - task-id: 1a-task
    type: structural | anomaly   # structural = expected config incompatibility; anomaly = runtime detection failure
    reason: "Commit strategy is not per-task; cannot identify fix commit to revert"
new-tests:                     # files critique created that should be committed
  - task-id: 1a-task
    file: src/tests/range-field.spec.ts
notes: |
  Overall approach is sound.
  Task 1b has two issues that need addressing.
```

Only `blocking` issues result in a `needs-work` verdict for the referenced task. Validation failures (test passes without fix or fails with fix) are blocking. Warnings and nits are recorded but do not block completion.

The critique agent must not modify source files. If it does, treat this as a bug and discard those changes. To detect this, the orchestrator should snapshot modified files (via `git status` or equivalent) before dispatching the critique agent and revert any newly modified files outside the task directories afterward.

#### Default gate prompt

When no custom `critique.prompt` is provided, no `validate-fix` tasks are present, and the level is not exclusively `explore` agents:

```
You are reviewing the work of a group of agents that ran in parallel. You have,
for each task in the level:

1. The task's plan.md (what was asked)
2. The task's output.yaml (what the agent claims it did)
3. The evidence files listed in the task's verification-summary

For each non-explore task, evaluate whether:
- The objective in plan.md was met
- The files listed in files-modified actually exist on disk and match what
  the plan described (check each file's contents against the plan's "Files to
  Modify" section and the task's stated objective)
- Constraints from the plan were respected
- Verification was actually performed:
  - For automated: Check verification.log shows tests/builds were run
  - For manual: Check manual-evidence.log demonstrates the task works
  - For review: Check verification.log shows compile/lint/tests were run
- Verification evidence matches the summary in output.yaml

For each explore task (agent: explore), evaluate whether:
- Referenced files and directories actually exist
- Function signatures, type definitions, and exports match what was reported
- Patterns described are accurate (spot-check against actual codebase)
- Important files or patterns were not missed
- The conclusions follow from the evidence in the codebase
- Check the deviations field in output.yaml; if the agent examined files not
  in the plan, a deviation is reasonable if the file was contextually necessary
  to answer the task's question (e.g., following an import chain); flag as
  blocking if the agent examined files that are unrelated to the objective

Important: Issues that must be marked as blocking:
- Objective not met
- A claimed file does not exist or does not contain the expected changes
- Verification not performed (missing or fake evidence)
- Verification evidence contradicts the summary
- Build/test/lint failures in evidence
- A finding that downstream tasks will rely on being wrong (explore tasks)

Write gate-critique.yaml to the dispatch run directory. Include:
- tasks: list of all task IDs you reviewed (e.g., [1a-task, 1b-task])
- verdict: accepted | needs-work
- issues: list of specific issues with task-id and severity (blocking | warning | nit)
- new-tests: entries (task-id + file) for any test files you created during review (omit if none)
- notes: optional free-text observations for the orchestrator
Only blocking issues should result in a needs-work verdict for a task.
```

#### Default gate prompt with validation

When no custom `critique.prompt` is provided and `validate-fix` is enabled for one or more tasks in the level:

```
You are reviewing the work of a group of agents with MUTATION TESTING enabled
for some tasks.

Follow the standard gate critique process for all tasks, then perform mutation
testing for each task that has validate-fix enabled:

MUTATION TESTING PROCEDURE (per task with validate-fix):
Note: mutation testing requires the `per-task` commit strategy. Each task must have committed its changes before the gate critique runs; otherwise there is no commit to revert. If the commit strategy is not `per-task`, skip mutation testing for this task and add an entry to `skipped-validation` in gate-critique.yaml with `type: structural` and a reason. Do NOT leave the `validation` entry absent — use `skipped-validation` so the orchestrator knows the skip was intentional.

Also note: `git log -1 -- <file>` returns the most recent commit that touched each file, not necessarily this task's commit. If downstream tasks or fix tasks have re-touched files after this task committed, the SHA lookup may return a later commit. The all-files-same-SHA check below detects the obvious case but cannot detect when all files were re-touched by the same later commit.

1. If `files-modified` is empty, add a `skipped-validation` entry with `type: structural` and reason "files-modified is empty" — do not attempt mutation testing.
2. Run `git log -1 --format="%H" -- <file>` for **each** file in that task's `files-modified`. If all files return the same SHA, use that SHA to revert. If any file returns a different SHA (i.e., different files were last touched by different commits), add an entry to `skipped-validation` with `type: anomaly` and reason "ambiguous commit attribution: files-modified span multiple commits" — do not attempt mutation testing for this task.
3. Revert the fix: git revert --no-commit <fix-commit-sha>
4. Run the test command: <test-command from validate-fix config>
5. Capture full output to: mutation-test-without-fix.log (in that task's directory)
6. Restore the fix: git revert --abort (aborts the in-progress revert; restores the working tree to HEAD in both clean and conflict cases; does not touch untracked new test files)
7. Run the test command again
8. Capture full output to: mutation-test-with-fix.log (in that task's directory)

VALIDATION CRITERIA:
- Test MUST fail when fix is reverted (without-fix status: failed)
- Test MUST pass when fix is present (with-fix status: passed)
- Any deviation is a BLOCKING issue for that task

WORKING TREE REQUIREMENTS:
- DO NOT make any git commits
- Working tree must be clean when finished (back to original state)
- Exception: new test files you create should remain

OUTPUT:
Write gate-critique.yaml to the dispatch run directory. Include:
- tasks: list of all task IDs you reviewed
- verdict: accepted | needs-work
- issues: list of specific issues with task-id and severity (blocking | warning | nit)
- validation: a list, one entry per task where mutation testing ran; each entry has task-id and mutation-test subfields (see gate-critique.yaml format)
- skipped-validation: a list, one entry per task where mutation testing was skipped; each entry has task-id, type (structural | anomaly), and reason (omit if none skipped)
- new-tests: entries (task-id + file) for any test files you created (omit if none)
- notes: optional free-text observations for the orchestrator
```

#### Default explore gate prompt

When all tasks in the gate input set are `explore` agents, no custom `critique.prompt` is provided, and no `validate-fix` tasks are present:

```
You are verifying the research findings of a group of explore agents that ran
in parallel. You have, for each task:

1. The task's plan.md (what was asked)
2. The task's output.yaml (the agent's findings)

Your job is to independently verify key claims against the actual codebase.
Do NOT take findings at face value.

For each task, check whether:
- Referenced files and directories actually exist
- Function signatures, type definitions, and exports match what was reported
- Patterns described (e.g., "all routes use X middleware") are accurate
- Important files or patterns were not missed
- The conclusions follow from the evidence in the codebase
- Check the deviations field in output.yaml; if the agent examined files not
  in the plan, a deviation is reasonable if the file was contextually necessary
  to answer the task's question (e.g., following an import chain); flag as
  blocking if the agent examined files unrelated to the objective

Write gate-critique.yaml to the dispatch run directory. Include:
- tasks: list of all task IDs you reviewed
- verdict: accepted | needs-work
- issues: list of specific issues with task-id and severity (blocking | warning | nit)
- notes: optional free-text observations for the orchestrator
Only blocking issues should result in a needs-work verdict for a task.
A finding that downstream tasks will rely on being wrong is blocking.
```

### Step 3c: Fix task resolution

After the level gate critique completes and tasks are marked `completed` or `fixing`, check every newly completed task: if it has a `fixes` field, transition each referenced original task's status from `fixing` to `completed` in the manifest. When `fixes` is a list, transition all referenced originals. Only apply this transition when the original task's current status is `fixing`.

This is what unblocks downstream tasks. Downstream tasks `depends-on` the original task ID, not the fix task. When the original transitions to `completed`, downstream tasks enter the ready set in the next iteration of Step 1.

Note: if a fix task itself is rejected by the gate critique, the fix task transitions to `fixing` and a new fix task is created (e.g., `1a-fix2`) with `fixes` pointing to the same original task (not to the previous fix task). The rejected intermediate fix task (`1a-fix1`) stays at `fixing` permanently -- this is expected and cosmetic, since nothing references it in a `fixes` field. Only the original task's status matters for unblocking downstream dependents.

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

### Task Fix Escalation (Execution Phase)

The orchestrator tracks escalation by counting the fix depth for a given original task: the number of tasks in the manifest whose `fixes` field references that original task's ID. For example, if `1a-fix1` and `1a-fix2` both have `fixes: 1a-extract_auth_module`, the fix depth is 2. When a fix task references multiple originals via `fixes`, escalation uses the maximum fix depth across all referenced originals.

- Depth 1-2: Fix silently, report progress in manifest
- Depth 3: Notify user that a task is proving difficult, explain what is failing
- Depth 4+: Ask user for guidance before continuing. Present the error, what has been tried, and options

This escalation policy applies to both verification failures and critique rejections. The counter is shared -- if critique produced fix1 and fix2, and then verification produces fix3, the user is notified at fix3 (depth 3) regardless of which mechanism created the fixes.

There is no hard retry limit. Escalation ensures the user is involved when things are not converging. Note: when a fix task is itself rejected by critique and a new fix task is created, both contribute to the fix depth. This is conservative -- it may over-count relative to the number of distinct fix attempts, but ensures the user is involved sooner rather than later.

### Plan Critique Escalation (Pre-Execution)

Plan critique iterations follow the same escalation pattern:

- **Iterations 1-2:** Calling agent fixes issues in-place, re-runs critique
- **Iteration 3:** Notify user that plan validation is proving difficult
- **Iteration 4+:** Ask user for guidance via question tool:
  - Proceed anyway (accept plan with known issues)
  - Consult greybeard for architectural guidance
  - Abort dispatch and start over
  - Manual review needed

**Spec Issue Exception:**
When spec issues (`type: spec`) are found:
- Pause and have user update the spec directly
- Reset iteration counter to 0 after spec update
- Restart Phase 1 with updated spec
- This takes precedence over the iteration escalation above

**Note:** If the critique agent crashes or produces invalid output at any iteration, block immediately and ask user for decision. Do not proceed with invalid critique results.

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
   - Fix tasks run normally
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

This section applies to fix tasks created by both critique rejection (Step 3b) and verification failure (Phase 5 fix loop).

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
- `fixes`: references the original planned task ID or a list of IDs (not previous fix tasks). When this fix task completes, the execution engine (Step 3c) transitions each referenced original task from `fixing` to `completed`. Use a list when a single fix addresses a cross-task interaction (e.g., mismatched interfaces between two tasks).
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
| `ask-once` (default) | Present the full commit plan (all commits with files and messages). User approves or adjusts, then all commits are created. |
| `ask-each` | Present each commit individually for approval. User can approve, edit the message, or skip. |
| `auto` | Commits are created without user interaction. Verification already passed. |

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
   - Total tasks run
   - Fix rounds needed (if any)
   - Critique rounds (if any)
   - Files modified across all tasks
   - Commits created
   - Verification results
3. Ask the user whether to keep or clean up the `dispatch/` directory

## Resumability

This section describes how to resume a dispatch run. **Note:** This logic is invoked AFTER the Checkpoint (presentation + confirmation) when resuming from an existing dispatch.yaml.

### When `$ARGUMENTS` points to an existing `dispatch.yaml`:

**First, run existing dispatch flow:**
1. Load and validate the dispatch
2. Analyze current state
3. Present status to user
4. Wait for confirmation

**Then, based on run status:**

**If status is `completed`:**
- Report previous results
- Exit (no further action needed)

**If status is `failed`:**
- Present failure summary at the Checkpoint
- Ask user whether to:
  - Retry failed tasks (reset to `pending`, set status to `in-progress`, proceed to Phase 4)
  - Abort (exit, leave dispatch as-is)

**If status is `pending`:**
- Re-run Phase 3 validation
- Present at Checkpoint
- Ask for approval to proceed
- If approved: set status to `in-progress`, proceed to Phase 4
- This handles interruption between plan creation and the run

**If status is `in-progress`:**
1. Re-validate remaining DAG (Phase 3)
2. Handle interrupted tasks based on status:
   - `completed`: skip
   - `dispatched`: check for `output.yaml`. If exists, process it; else reset to `pending`
   - `failed`: present at Checkpoint, ask to retry or skip
   - `fixing`: check associated fix tasks:
     - `dispatched`: check for `output.yaml`
     - `completed` but parent still `fixing`: transition parent to `completed`
     - `failed`: present at Checkpoint, ask to retry or skip
     - `pending`: leave as-is
     - No fix task exists: treat original as `failed`
   - `pending`: proceed normally
3. After handling interrupted tasks, proceed from Phase 4, Step 1

**If `results.commits` is non-empty but status is `in-progress`:**
- Interruption occurred during Phase 6
- Check which commits exist in git log
- Skip existing commits
- Continue committing remaining tasks
- If all commits exist: proceed directly to Phase 7

### Clean Resume Requirements

For a clean resume, ensure:
- Working tree has no unrelated modifications
- All `output.yaml` files in `dispatched` tasks are valid
- No external changes to task directories

The orchestrator does not detect external changes between interruption and resume. Phase 5 verification will catch build/test/lint issues, but unrelated modifications may be silently included in commits.

## Source of Truth Contract

The orchestrator and subagents agree on the following:

| Artifact | Role | Who writes | Who reads |
|---|---|---|---|
| `plan.md` | Task instructions | Orchestrator | Subagent |
| `output.yaml` | Task results (source of truth) | Subagent (Orchestrator for `explore` agents) | Orchestrator |
| `level<N>-gate-critique.yaml` | Gate verdict for planned tasks at level N | Critique agent | Orchestrator |
| `level<N>-fix-round<R>-gate-critique.yaml` | Gate verdict for fix round R at level N | Critique agent | Orchestrator |
| Task tool return | Diagnostic supplement (advisory) | Subagent | Orchestrator (diagnostics only) |
| `dispatch.yaml` | Run state and DAG | Orchestrator | Orchestrator |
| `mutation-test-*.log` (in task dir) | Mutation testing evidence | Critique agent | Human (paths stored in gate-critique.yaml for reference; orchestrator never reads directly) |
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
