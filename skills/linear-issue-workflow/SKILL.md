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

## Phase 2: Set Up Worktree

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

All subsequent work — exploration, planning, implementation, and review — happens in the worktree directory.

## Phase 3: Explore and Plan

1. Use the `explore` subagent to understand the codebase. Brief it with the absolute path to the worktree (it must `cd` there before doing anything else), and ask it to cover:
   - Where changes need to be made
   - Existing patterns to follow
   - Related code that might be affected

2. Create an implementation plan covering:
   - Files to modify
   - New functions/types to add
   - Tests to write

3. Present the plan to the user and ask if they would like any changes before proceeding. Do not start implementation until the user approves the plan.

   If the user rejects the plan and the issue cannot be salvaged with a revised plan, tear down the worktree and branch with the Phase 7 commands rather than leaving them stranded.

4. Attach the plan as a file to the Linear issue. **Do not post the plan as a comment** — comments are for discussion, not archives, and a multi-page plan dumped inline creates noise on the issue.

   The procedure has four steps, three of which are MCP or HTTP calls. The signed upload URL from `prepare_attachment_upload` expires 60 seconds after it is returned, so steps 2 and 3 must happen in immediate succession — do not pause for unrelated work between them.

   1. Write the plan to a file under the worktree's `tmp/` (e.g., `tmp/plan-<ISSUE-ID>.md`). `tmp/` is the project's throwaway directory — do not commit the file (and if the project does not yet have `tmp/` gitignored, do not add to git). Capture its byte size with `wc -c < tmp/plan-<ISSUE-ID>.md`.

   2. Call `mcp__linear__prepare_attachment_upload` with `issue`, `filename`, `contentType: "text/markdown"`, and `size`. The response contains `uploadRequest.url`, `uploadRequest.headers`, and `assetUrl`. **Step 3 must follow within 60 seconds — the signed URL expires.**

   3. PUT the raw file bytes to `uploadRequest.url`. Pass every header from `uploadRequest.headers` verbatim — exact casing is required; modified or omitted headers return HTTP 403. One `-H` flag per entry:

   ```bash
   curl -X PUT --data-binary @tmp/plan-<ISSUE-ID>.md \
     -H "<header1-name>: <header1-value>" \
     -H "<header2-name>: <header2-value>" \
     "<uploadRequest.url>"
   ```

   Do not base64-encode the file, do not transform it. If the PUT fails with 403 because the URL expired (more than 60 seconds since step 2), **call `prepare_attachment_upload` again for a fresh URL** — retrying the dead URL will keep failing.

   4. Call `mcp__linear__create_attachment_from_upload` with `issue`, the `assetUrl` from step 2, `title: "Implementation plan"`, and `subtitle: "<ISSUE-ID>"` (so the entry stays identifiable if multiple plans accumulate across re-plans).

   Keep the local file through implementation and review — it is the working reference, colocated with the code in the worktree's `tmp/`. Phase 7's `git worktree remove` deletes it along with the rest of the worktree. The Linear attachment is the canonical copy; the local file is a working convenience for the active branch. Re-planning overwrites the local file — Linear retains prior versions as separate attachments via the `<ISSUE-ID>` subtitle.

   Exception: plans with no structure — no file-by-file breakdown, no enumerated steps, no headings, no nested lists — can be posted as a comment instead. Structural shape, not length, is the test; a single-sentence plan is fine inline, a 10-line bulleted plan is not.

5. Mark the issue as "In Progress" using `mcp__linear__save_issue` with the appropriate state.

## Phase 4: Implement

Carry out the plan from Phase 3 in the worktree using whichever implementation approach fits the work and your role.

Break the work into commit-sized units. Each commit should represent one logical change that can be reviewed and understood independently.

### Keep the issue checkboxes current

If the issue description contains a task list (`- [ ]` items), tick the boxes as you complete each one. The checkboxes are the issue's at-a-glance progress signal — leaving them stale makes the issue lie about what's done.

Update the description with `mcp__linear__save_issue`, passing the full description with the relevant `- [ ]` flipped to `- [x]`. Do not rewrite or reorganize the surrounding text — only flip the box. Update as each item finishes, not in a single batch at the end; partial progress is what the checkboxes exist to show.

If the issue description has no task list, skip this step — do not invent one.

## Phase 5: Self-Review

Once implementation is complete, run a **whole-branch code review** before pushing. The review covers the entire diff from the base branch to HEAD — not just the most recent commit — so any finding anywhere in the branch's history is in scope.

### Reviewer-of-Record Checks (in-session)

The reviewer-of-record is the agent whose verdict ships — in this workflow, you, the orchestrator. Before delegating any part of the review, run the audits whose value depends on the reviewer-of-record's own eyes on raw output. The canonical command list and patterns live in `code-review`'s *Reviewer-of-Record Checks* and *Commit-Message Style Audit* sections; load `code-review` and follow them in this session.

Read the raw output. Stop conditions — fix before continuing:

- Any violation surfaced by the commit-message audits. The canonical pattern lists (prefixes, vague subjects, body issues, length limits) live in `style` and `code-review`'s *Commit-Message Style Audit*; this section does not maintain a copy.
- `Bin` marker on any file you did not expect to be binary (source code, markdown, config).
- Files in the diff outside the issue's scope.

After fixing a stop condition, re-run the in-session checks. Repeat until the output is clean. Do not dispatch the subagent until then.

### Subagent review (deeper read)

Dispatch the `critique` subagent for the file-by-file behavioral read, architectural review, and commit-message coherence check. Running these in a subagent keeps the deeper output out of the main context and gives independent eyes on patterns.

Brief the subagent with:

- The absolute path to the worktree (it must `cd` there before doing anything else).
- The base branch to diff against (the remote default branch resolved in Phase 2, e.g. `origin/main`), so `code-review` does not dead-end on its "ask the user" path. Make explicit that the review must cover the full `base..HEAD` range, every commit on the branch.
- An instruction to load the `code-review` skill and follow its checklist against the current branch, *except* the items marked *(reviewer-of-record)* — the orchestrator has already run those.
- The Linear issue ID and a one-line summary of the change's *intent* — what the change is for. "This branch refactors retry logic to use exponential backoff" is fine; "this branch should not introduce blocking calls in the sendPack path" pre-frames findings and is forbidden.
- A request for a punch list of findings with `file:line` references — not PR-comment prose.
- An instruction that the subagent's final message must be the full punch list verbatim, with no summarization or omission. This prevents collapse during transmission; it does *not* prevent the more fundamental loss of signals that do not fit a punch-list shape at all (binary markers, surprising stat counts). Those belong to the reviewer-of-record checks above.

Do **not** include author-supplied "blocking criteria" or "things to look for" in the brief. The skill itself is the criteria; supplementing it narrows the subagent's lens to what you already suspect might be wrong, suppressing unknown-unknowns. The intent summary above is bounded for the same reason — keep it to *what the change is for*, never *what the reviewer should find*.

### Fix every finding

Treat the returned findings as a worklist and **fix every one**. The `code-review` skill's "Signal Over Noise" guidance has already filtered out pedantic taste-only nitpicks upstream — anything that survived to the punch list is something the reviewer judged worth the author's time. "Nit," "minor," "stylistic," and "suggestion" describe the reviewer's confidence about severity; they are not dispositions and do not authorize skipping. The only path to leaving a finding unfixed is a Greybeard waiver (see below).

- Fix issues in additional commits, or via `git rebase -i` with `edit` on the target commit when a fix belongs on an earlier commit (mark the target `edit`, make the fix at the stop, `git commit --amend --no-edit`, `git rebase --continue`).
- After fixing, re-run the reviewer-of-record checks in-session, then re-dispatch the review subagent against the full branch as it now stands. Every re-review is a fresh, self-contained read of `base..HEAD` exactly as it is — no scoping to the delta from a prior pass, no carryover ledger of earlier findings. The new punch list is the only worklist; the previous one is gone.
- Repeat fix-and-review until the review returns clean.
- Cap at three re-reviews. The cap counts subagent dispatches; in-session check failures the orchestrator fixes between dispatches do not consume a slot. If findings remain after the third, stop and surface the situation to the user directly. Do not soften it: tell the user plainly that the branch has been through three review-and-fix passes and the reviewer is still finding issues, list each outstanding finding with its `file:line`, and state explicitly that the branch is **not ready to push**. Do not proceed to Phase 6, do not propose a waiver, and do not offer to "just push anyway" — wait for the user to decide whether to keep iterating, rescope the issue, or abandon the branch.

### Waivers

The only path to leaving a finding unfixed is a Greybeard waiver. If you believe a finding should not be fixed — because the "fix" would be pure churn (taste-only rewording, unrelated refactor, change the project has explicitly chosen not to make) or because you actively disagree with the reviewer — dispatch the `greybeard` subagent with the finding, your proposed disposition, and the relevant diff. **A Greybeard "waive" ruling is the waiver** — record it in the disposition note and move on; no further user sign-off is needed. Accept Greybeard's call by default. Escalate to the user only if (a) you actively disagree with Greybeard's ruling, or (b) the Greybeard subagent is unreachable or returns an unusable response. In either case, present both positions (or the failure mode) and let the user decide. Do not route routine waiver requests through the user, and never waive a finding on your own authority.

The full punch list returned by the review subagent (and, if a re-review happened, the carried-over plus newly-reported findings, deduplicated) is what gets posted to the PR in Phase 6. Keep it intact.

## Phase 6: Push and PR

Once the Phase 5 gate has cleared (the review returned clean, or every remaining finding has a Greybeard waiver), ask the user for confirmation before pushing and creating the PR.

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

After the PR is created, post the review output captured in Phase 5 as a PR comment, with a brief note on how each finding was addressed (fixed or Greybeard-waived):

```bash
gh pr comment --body "$(cat <<'EOF'
## Self-Review (`code-review` skill)

<verbatim review output from Phase 5>

### Disposition

- <finding>: <fixed in <sha> | waived (Greybeard, <one-line ruling>)>
EOF
)"
```

If the review came back clean, post a one-line comment saying so rather than skipping the step — the absence of findings is itself useful signal for reviewers.

## Phase 7: Cleanup After Merge

After the PR has been merged — or after a Phase 3 plan rejection that abandons the issue — clean up the worktree and local branch. The orchestrator is still `cd`'d into the worktree from Phase 2; leave it before running these commands:

```bash
cd <path-to-main-repo>           # leave the worktree
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

| Action          | Tool                                                                                              |
| --------------- | ------------------------------------------------------------------------------------------------- |
| Fetch issue     | `mcp__linear__get_issue`                                                                          |
| Get branch name | `mcp__linear__get_issue` (read `branchName` from result)                                          |
| Update status   | `mcp__linear__save_issue` (set the state)                                                         |
| Add comment     | `mcp__linear__save_comment`                                                                       |
| Attach file     | `mcp__linear__prepare_attachment_upload` → PUT → `mcp__linear__create_attachment_from_upload` (see Phase 3 step 4) |
| List teams      | `mcp__linear__list_teams`                                                                         |
