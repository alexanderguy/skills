---
name: code-review
description: Perform a code review or pull request review on a branch
disable-model-invocation: true
---

# Code Review

Use this skill when performing code reviews or pull request reviews.

## Scope Determination

Focus only on commits that are contained within the branch being reviewed.
Use `git diff <base-branch>...HEAD` to determine what is in scope.

```bash
# List commits on the branch
git log --oneline <base-branch>..HEAD

# Get the full diff for review
git diff <base-branch>...HEAD

# Get diff for a specific file
git diff <base-branch>...HEAD -- <file>
```

## Pre-existing Code

If a bug, convention violation, or inconsistent naming existed in code before
the branch, do not treat it as a problem in the review. This includes cases
where refactored code retains a pre-existing type name, variable name, or
pattern that does not match current conventions.

Only evaluate the changes introduced by the branch.

## Convention Compliance

All new logic, patterns, and naming introduced by the branch should follow
the project's established conventions. Pre-existing code that appears in
the diff due to refactoring is exempt from this requirement.

If reviewing TypeScript code, consider loading the `typescript-conventions`
skill for detailed guidance on type patterns, naming, and idioms.

## Delegating to Sub-agents

When delegating file review to sub-agents, provide the output of
`git diff <base-branch>...HEAD -- <file>` rather than the full file contents.

Sub-agents that receive full files cannot distinguish branch changes from
pre-existing code and will flag out-of-scope issues.

If full files must be provided for context, explicitly instruct the sub-agent
which line ranges were modified by the branch and that only those ranges are
in scope.

## Test Coverage Philosophy

Do not request additional test coverage for functionality provided by external
libraries. Focus test coverage requests on:

- Business logic and domain-specific validation
- Integration points between components
- Error handling paths and edge cases
- Custom algorithms and data transformations

Trust well-maintained libraries to do their job. If a library's behavior needs
testing, that suggests reconsidering whether to use that library.

## Review Checklist

1. Determine the base branch (usually `main`)
2. Run `git log --oneline <base>..HEAD` to understand the scope
3. Run `git diff <base>...HEAD --stat` to see which files changed
4. Review each changed file, focusing only on lines modified by the branch
5. Check that new code follows project conventions
6. Summarize findings with specific file:line references
