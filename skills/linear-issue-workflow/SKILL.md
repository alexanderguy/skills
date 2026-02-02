---
name: linear-issue-workflow
description: Implement a feature or fix based on a Linear issue
argument-hint: "<issue-id> [--reviewer <reviewer>]"
---

# Linear Issue Workflow

Use this skill when implementing features or fixes tracked in Linear.

## Phase 1: Understand the Issue

Fetch the Linear issue using the `linear` subagent:

```
Task(subagent_type="linear", prompt="Fetch issue <ISSUE-ID> including title, description, status, and acceptance criteria")
```

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

4. Optionally attach the plan to the Linear issue:

   ```
   Task(subagent_type="linear", prompt="Attach this implementation plan as a document to <ISSUE-ID>: <plan>")
   ```

5. Mark the issue as "In Progress":
   ```
   Task(subagent_type="linear", prompt="Update issue <ISSUE-ID> status to In Progress")
   ```

## Phase 3: Set Up Worktree

Get the branch name from Linear:

```
Task(subagent_type="linear", prompt="Get the git branch name for issue <ISSUE-ID>")
```

Create a worktree to keep the main repo clean:

```bash
git worktree add ../worktree/<branch-name> -b <branch-name>
```

All subsequent work happens in the worktree directory.

## Phase 4: Implement

1. Use `TodoWrite` to track implementation tasks
2. Implement changes, marking todos as complete as you go
3. Write tests for new functionality
4. Run the project's build/verification command

## Phase 5: Verify

If the changes affect critical paths or integration points, run the project's test suite or integration tests as appropriate for the changes made.

## Phase 6: Commit and PR

### Squash commits if needed

If you made multiple commits that should be one:

```bash
git reset --soft HEAD~<n> && git commit -m "<message>"
```

### Self-review

Before pushing, load the `code-review` skill and perform a self-review of your changes. If serious issues are found that would prohibit merging, fix them before proceeding. Do not push until the review passes.

After the review passes, ask the user for confirmation before pushing and creating the PR.

### Push and create PR

```bash
git push -u origin <branch-name>

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

## Linear Subagent Patterns

Common operations:

| Action          | Prompt                                                          |
| --------------- | --------------------------------------------------------------- |
| Fetch issue     | `Fetch issue <ISSUE-ID> with full details`                      |
| Get branch name | `Get the git branch name for issue <ISSUE-ID>`                  |
| Update status   | `Update issue <ISSUE-ID> status to In Progress`                 |
| Attach document | `Attach this as a document titled "X" to <ISSUE-ID>: <content>` |
| Add comment     | `Add a comment to <ISSUE-ID>: <comment>`                        |

Always use `subagent_type="linear"` when calling the Task tool for Linear operations.
