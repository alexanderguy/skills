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

Load the `implement` skill and follow its workflow for each logical unit of work.

The implementation plan from Phase 2 defines what to build. Break it into commit-sized units and work through each one using the implement skill's per-commit discipline: Greybeard review, implement and test, build gate, commit, Critique loop.

Keep commits focused. Each commit should represent one logical change that can be reviewed and understood independently.

## Phase 5: Push and PR

After all commits are complete, ask the user for confirmation before pushing and creating the PR.

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

## Phase 6: Cleanup After Merge

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
