---
name: linear-issue-workflow
description: Implement a feature or fix based on a Linear issue
argument-hint: "<issue-id> [--reviewer <reviewer>]"
disable-model-invocation: true
---

# Linear Issue Workflow

Use this skill when implementing features or fixes tracked in Linear.

## Phase 1: Understand the Issue

Fetch the Linear issue using `mcp__linear__get_issue`. The returned issue includes title, description, status, branch name, and other metadata you will need later.

Ask the user clarifying questions if the scope is unclear before proceeding.

## Phase 2: Explore and Plan

1. Use the `explore` subagent to understand the codebase:
   - Where changes need to be made
   - Existing patterns to follow
   - Related code that might be affected

2. Create an implementation plan covering:
   - Files to modify
   - New functions/types to add
   - Tests to write

3. Present the plan to the user and ask if they would like any changes before proceeding. Do not start implementation until the user approves the plan.

4. Optionally attach the plan to the Linear issue as a comment using `mcp__linear__save_comment`.

5. Mark the issue as "In Progress" using `mcp__linear__save_issue` with the appropriate state.

## Phase 3: Set Up Worktree

Read the branch name from the `branchName` field on the issue fetched in Phase 1 (call `mcp__linear__get_issue` again if needed).

Determine the repository's default branch:

```bash
git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'
```

Fetch the latest changes and create a worktree based on the remote default branch:

```bash
git fetch origin
git worktree add ../worktree/<branch-name> -b <branch-name> origin/<default-branch>
```

**Important**: Always base new branches on `origin/<default-branch>` (where `<default-branch>` is `main`, `master`, or whatever the repository uses). This ensures your branch starts from the latest remote state, even if your local default branch is out of date.

After creating the worktree:

```bash
cd ../worktree/<branch-name>
```

Then install local dependencies (look at developer documentation for how to do this).

Worktrees share the `.git` directory but have their own working directory. The `node_modules` directory is NOT shared, so each worktree needs its own dependency installation.

All subsequent work happens in the worktree directory.

## Phase 4: Implement

Carry out the plan from Phase 2 in the worktree using whichever implementation approach fits the work and your role.

Break the work into commit-sized units. Each commit should represent one logical change that can be reviewed and understood independently.

## Phase 5: Self-Review

Before pushing, dispatch the `critique` subagent to review the branch using the `code-review` skill. Running the review in a subagent keeps its full output out of the main context and gives an independent read of the diff.

Brief the subagent with:

- The absolute path to the worktree (it must `cd` there before doing anything else).
- The base branch to diff against (the remote default branch resolved in Phase 3, e.g. `origin/main`), so `code-review` does not dead-end on its "ask the user" path.
- An instruction to load the `code-review` skill and follow its checklist against the current branch.
- The Linear issue ID and a one-line summary of the intended change, so the reviewer can judge scope.
- A request for a punch list of findings with `file:line` references — not PR-comment prose.
- An instruction that the subagent's final message must be the full punch list verbatim, with no summarization or omission. The main agent will only see what the subagent returns; anything left out is lost.

Treat the returned findings as a worklist:

- Fix issues in additional commits, or via `git commit --fixup=<sha>` + `git rebase --autosquash` when a fix belongs on an earlier commit.
- Re-dispatch the review subagent only after a fix round, and brief it to focus on the files touched since the previous review (pass the changed paths explicitly). Findings on untouched files from the previous round carry over without re-review.
- Cap re-reviews at three rounds. If major findings remain after the third round, stop and surface the situation to the user directly. Do not soften it: tell the user plainly that the branch has been through three review-and-fix cycles and the code still has unresolved major issues, list each outstanding finding with its category (correctness / safety / scope / build-test / convention) and `file:line`, and state explicitly that the branch is **not ready to push**. Do not proceed to Phase 6, do not propose a waiver, and do not offer to "just push anyway" — wait for the user to decide whether to keep iterating, rescope the issue, or abandon the branch.

### Gate

Major findings block Phase 6. A finding is **major** if any of the following hold:

- Correctness: the change is buggy, breaks existing behavior, or fails to do what the Linear issue asked for.
- Safety: introduces a security issue, swallows errors, removes a defensive check, or risks data loss.
- Scope: includes substantive changes unrelated to the issue, or omits something the issue explicitly required.
- Build/test: leaves the build or test suite broken.
- Convention: violates a project convention in a way that would be rejected on human review.

Do not push until every major finding is either fixed (re-run the review to confirm the pattern is gone from the diff) or waived. Minor findings (nits, stylistic preferences, suggestions) do not gate the push, but you should still fix any that are material improvements — clearer naming, removing dead code, tightening a confusing branch, etc. Only drop a minor finding to the disposition note if fixing it would be pure churn (taste-only rewording, unrelated refactors, or changes the project has explicitly chosen not to make).

Greybeard is the arbiter for judgment calls. Whenever you are unsure whether a finding is major, want to waive one, or want to disagree with the reviewer, dispatch the `greybeard` subagent with the finding, your proposed disposition, and the relevant diff. **A Greybeard "waive" ruling is the waiver** — record it in the disposition note and move on; no further user sign-off is needed. Accept Greybeard's call by default. Escalate to the user only if (a) you actively disagree with Greybeard's ruling, or (b) the Greybeard subagent is unreachable or returns an unusable response. In either case, present both positions (or the failure mode) and let the user decide. Do not route routine judgment calls through the user.

The full punch list returned by the review subagent (and, if a re-review happened, the carried-over plus newly-reported findings, deduplicated) is what gets posted to the PR in Phase 6. Keep it intact.

## Phase 6: Push and PR

Once the Phase 5 gate has cleared (all major findings either fixed or waived), ask the user for confirmation before pushing and creating the PR.

### Push and create PR

Fetch the latest changes and rebase before pushing:

```bash
git fetch origin
git rebase origin/<default-branch>
```

Verify the build still passes after rebasing, then push:

```bash
git push -u origin <branch-name>
```

Create the PR:

```bash
gh pr create \
  --title "<title>" \
  --reviewer <REVIEWER> \
  --body "$(cat <<'EOF'
## Summary

<1-3 bullet points>

## Changes

- `path/to/file.ts`: <what changed>

## Testing

<how it was tested>

Closes <ISSUE-ID>
EOF
)"
```

### Post the self-review

After the PR is created, post the review output captured in Phase 5 as a PR comment, with a brief note on how each finding was addressed (fixed, deferred, disagreed):

```bash
gh pr comment --body "$(cat <<'EOF'
## Self-Review (`code-review` skill)

<verbatim review output from Phase 5>

### Disposition

- <finding>: <fixed in <sha> | deferred — <reason> | disagreed — <reason>>
EOF
)"
```

If the review came back clean, post a one-line comment saying so rather than skipping the step — the absence of findings is itself useful signal for reviewers.

## Phase 7: Cleanup After Merge

After the PR has been merged, clean up the worktree and local branch:

```bash
# From the main repository directory (not the worktree)
git fetch origin
git worktree remove ../worktree/<branch-name>
git branch -d <branch-name>
```

If the worktree directory was already manually deleted, prune stale worktree references:

```bash
git worktree prune
```

## Linear MCP Tool Reference

Common operations:

| Action          | Tool                                                       |
| --------------- | ---------------------------------------------------------- |
| Fetch issue     | `mcp__linear__get_issue`                                   |
| Get branch name | `mcp__linear__get_issue` (read `branchName` from result)   |
| Update status   | `mcp__linear__save_issue` (set the state)                  |
| Add comment     | `mcp__linear__save_comment`                                |
| List teams      | `mcp__linear__list_teams`                                  |
