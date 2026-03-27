---
name: implement
description: Disciplined per-commit workflow with Greybeard review, build gates, and Critique loops
---

# Implement

A disciplined implementation workflow that produces reviewed, verified commits. Load this skill when you want each commit to go through architectural review, build verification, and code critique before it lands.

## Prerequisites

Before using this workflow, load the `style` and `philosophy` skills. Follow their conventions throughout.

## When to Use

This is a standalone skill, loaded on request. It is not part of dispatch. Use it when you want a single agent to work through a series of commits with review discipline.

The caller defines what work to do and where the commit boundaries are. This skill defines *how* each commit gets produced.

## Workflow Per Commit

For each logical unit of work that results in a commit, follow these steps in order. Do not skip steps.

### Step 1: Greybeard Review

Before writing any code, describe your implementation approach to Greybeard and ask for feedback.

**What to send Greybeard:**
- What you're about to change and why
- Which files you expect to touch
- Any design decisions or trade-offs you're considering
- Anything you're uncertain about

**How to handle feedback:**
- If Greybeard identifies problems with your approach, adjust before proceeding
- If Greybeard suggests a fundamentally different approach, consider it seriously
- You don't need to agree with every suggestion, but you need a reason to disagree
- Once you're aligned on approach, move to Step 2

Use the `@greybeard` subagent for this step.

### Step 2: Implement

Make the changes. Follow the approach reviewed in Step 1.

Keep the scope tight to what was discussed. If you discover additional work is needed, finish the current commit's scope first and note the additional work for a future commit.

### Step 3: Build Gate

Run `make` (or the project's equivalent full pipeline: format, lint, build, test).

- If the build passes, move to Step 4
- If the build fails due to your changes, fix the failures and re-run until it passes
- If the build fails due to pre-existing issues unrelated to your changes, report the failure to the caller and let them decide how to proceed
- Do not move forward with a broken build
- Do not substitute partial builds (e.g., running only the compiler) for the full pipeline

### Step 4: Commit

Create the commit. Follow the commit message conventions from the `style` skill.

### Step 5: Critique Loop

Ask Critique to review the committed change.

**How to run:**
1. Spawn the `@critique` subagent and ask it to review the output of `git show HEAD`. Include the intent from Step 1 (what the change is meant to accomplish and the approach agreed with Greybeard) so Critique can evaluate whether the implementation matches the plan, not just surface-level quality. Tell Critique to limit its findings to the scope of the current commit -- pre-existing issues in touched files are out of scope.
2. Read its findings
3. For each issue marked VERIFIED or HIGH confidence: fix it
4. Re-run the build gate (Step 3) to verify fixes
5. Amend the commit with the fixes. If a fix logically belongs on an earlier unpushed commit rather than HEAD, use a fixup commit per the `style` skill's guidance.
6. Ask Critique to review `git show HEAD` again. Re-include the original intent from Step 1 and tell it what you fixed since the last pass so it can focus on verifying the fixes and checking for new issues rather than re-reviewing the entire change from scratch.
7. Repeat until Critique comes back clean or all remaining findings are acknowledged and intentional

**When to stop looping:**
- Critique reports no issues
- Remaining findings are judgment calls you've consciously decided against, not oversights
- The build passes after the last round of fixes

### Step 6: Next

Move to the next unit of work. Return to Step 1.

## Guidelines

**Don't shortcut the loop.** The value is in the discipline. Skipping Greybeard "because this change is simple" or skipping Critique "because the build passes" defeats the purpose.

**Keep commits focused.** If Critique finds something wrong outside the scope of the current commit, note it for a later commit rather than expanding scope mid-loop.

**Build must pass before every commit and amend.** Never commit code that doesn't compile or pass tests. Fix build failures first, then commit or amend.

**Greybeard is for approach, Critique is for execution.** Greybeard reviews your plan before you write code. Critique reviews your code after you write it. Don't conflate the two.

## Acknowledgment

After reviewing this skill, state: "I have reviewed the implement skill and am ready to follow the commit workflow."
