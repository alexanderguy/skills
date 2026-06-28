---
name: bug-triage
argument-hint: <bug report — a Linear/GitHub issue ID or URL, a file path, or pasted text>
description: Validate a filed bug against the source before fixing it — verify every claim, hunt the broader class, find the contract question, and scope the outcome
---

# Bug Triage

Use this skill to take a filed bug (or a "repro for X" ticket) and turn it into
a *trustworthy* understanding of the problem, a verified fix direction, and
correctly-scoped follow-up work — instead of trusting the report and shipping a
one-liner that relocates the failure.

The core stance: **the report is a hypothesis, the code is the truth.** A bug
report is written by someone (or some agent) reconstructing a failure from
partial evidence. It is frequently right about the symptom, often wrong about a
load-bearing detail, and almost always narrower than the real problem. Treat
every claim as unverified until you have read the code that proves or refutes
it.

This skill is about *validating and scoping* a bug. It stops at a defensible
understanding and a correctly-split plan. It does not implement the fix — hand
the resulting correctness fix to the `implement` skill, and file the broader
work with `linear-create`.

## Input

The bug report arrives as `$ARGUMENTS` — a Linear or GitHub issue ID/URL, a path
to a file, or pasted text. Get the full report in front of you before Phase 1:

- A Linear issue → fetch it with the Linear MCP tools (`get_issue`, plus
  `list_comments` for discussion). Call these from the main conversation, not a
  subagent.
- A GitHub issue/PR URL → fetch it with `gh issue view` / `gh pr view`.
- A file path or pasted text → read it directly.

If the report references a linked or related issue (Phase 1), fetch that too.

## When to use this skill — and when not to

Use the full process for a report whose load-bearing claims you cannot confirm
cheaply yourself: anything touching a contract or lifecycle question, a proposed
fix that changes a wire format or schema, a sweep that returned a suspicious
count, or a symptom that smells like one instance of a class.

Scale it down — or skip it — when the bug is small, obvious, and verifiable in
seconds: a typo, an off-by-one with the failing line right in front of you, a
crash with a stack trace pointing at the exact statement. Forcing a verifier
sweep, a thorough class-hunt, and a greybeard review through a one-line typo
burns effort and trust for nothing; "pragmatic over idealistic" (`philosophy`)
governs here. Collapse the phases to the ones that earn their keep, and reach for
subagents only when the reading genuinely exceeds what you can do inline.

## Prerequisites

Before triaging anything, load the `style` and `philosophy` skills. They are not
background reading — they are the active constraints any fix you propose must
satisfy, and "Constraint Ownership" in `philosophy` is the lens for the
contract question in Phase 6. If the repository has its own `AGENTS.md`,
`CONTRIBUTING`, or `CONVENTIONS` docs, read those too.

## Failure modes this prevents

These recur whenever a report is taken at face value:

- **Wrong specifics carried downstream.** A report says "503"; the code returns
  502. It names trigger functions that don't exist in the repo (they live in a
  separate vendored consumer). It claims "zero usages of X" when there is one.
  Each wrong fact sends the next engineer hunting the wrong path.
- **Myopia.** The filed bug is usually *one instance of a class*. Fixing only the
  reported site leaves the siblings to be re-filed next week.
- **The relocating fix.** The proposed one-line fix often doesn't remove the
  failure — it moves it later, deeper, or into an adjacent code path, where it is
  harder to diagnose. (An upsert that makes one insert idempotent inside a
  multi-step operation that isn't; a "register early" that trades a startup error
  for silent mail loss.)
- **The unasked question.** The hardest bugs hide a contract or design question
  the ticket never asks — "what does a relaunch *mean*?", "what does *ready*
  mean, and ready for *what*?" The one-line fix silently answers it, usually
  wrong.
- **Count inflation.** An automated sweep returns "55 issues"; most are the same
  bug restated, or "code lacks guard X" miscoded as "vulnerable." Acting on the
  raw count wastes effort and erodes trust.

## The process

Phase 1 comes first — it produces the claim checklist everything else verifies
against. Once it exists, the *reading* in Phases 2, 3, and 6 is independent
legwork that can run in parallel. Reserve your own reasoning for the judgment in
Phases 4 and 8, and for the *call* in Phase 6 (its evidence-gathering
parallelizes; the verdict is yours). The `dispatch` skill is a good fit when you
want to fan the legwork out and gate on the results.

### Phase 0 — Ground yourself in the repo's rules

Load `style`, `philosophy`, and any repo-local convention docs (`AGENTS.md`,
`CONTRIBUTING`, `CONVENTIONS`) before judging anything — the Prerequisites
section above covers this. You cannot tell a real violation from a house-style
choice without them.

Also locate the **build and test entrypoints** now (how the project builds, how
its tests run, where they live). Phase 8 calls for a repro test, and you cannot
scope one without knowing how tests are written and run here.

### Phase 1 — Read the report as a set of claims

Extract every *load-bearing factual claim* the report makes, separately from its
narrative, and write them down as a checklist:

- Specific file:line references and quoted code
- The error/status code and the exact error string
- The trigger condition ("only when X reuses Y with at least one Z")
- The named functions and call paths
- The proposed fix and the report's stated reason for rejecting alternatives
- Linked or related issues and what they assert

This checklist is what you will verify. Nothing in the narrative gets to skip it.

### Phase 2 — Verify every claim against source (adversarial)

Dispatch a **skeptical verifier** whose job is to confirm or refute each claim
with file:line evidence and quoted snippets — *not* to agree with the report.
Use a `general-purpose` subagent: rendering a verdict, and especially judging
reachability, is analysis, not search, so the read-only `Explore` agent (which
locates code but does not audit it) is the wrong tool for the call. `Explore` can
still feed this phase raw file:line evidence; the verdict is the verifier's.
Instruct the verifier explicitly:

- "Do not trust the report. Read the code. Report what is actually true."
- "Mark each claim CONFIRMED / PARTIALLY CONFIRMED / REFUTED."
- "Where line numbers have drifted or a name is wrong, say so."
- "Distinguish *the unsafe-looking code exists* from *the unsafe path is
  reachable*." This one distinction repeatedly turns a "critical bug" into a
  non-issue — e.g. a guard that looks skippable but cannot execute because
  subscription is guaranteed before the socket accepts traffic.

Output: a per-claim verdict table. Now you know what is real.

### Phase 3 — Hunt the broader class (assume the report is myopic)

In parallel, dispatch an **exploration** for *other instances of the same
underlying disease*. A `very thorough` `Explore` sweep is the right tool. Frame
it by the disease, not the symptom: not "find other places that insert into
table T" but "find operations assumed to run once that can run again"; not
"other 503s" but "other places a component signals ready before it is usable."

Give the sweep explicit categories to hunt, and ask for **file:line, the
trigger, and severity** per finding — and to name any existing helper or pattern
that *should* be reused, so a fix can be centralized rather than sprinkled.

### Phase 4 — Triage the class with judgment (don't relay the sweep)

A sweep produces raw findings, not conclusions. Apply a senior filter yourself:

- **Collapse restatements.** Six findings that are the same defect seen from six
  angles are one bug. Say so.
- **Reject miscoded severity.** "Lacks guard X" is only a bug if (a) a real
  constraint exists *and* (b) a path actually re-triggers it. A log line does not
  "drop traffic." A random-keyed insert that re-runs does not collide — it
  orphans, a *different* problem.
- **Promote the genuinely distinct siblings** — especially ones *worse* than the
  filed bug, or pointing the opposite direction (the filed bug was "registered
  too late"; a sibling is "routable too early").
- **Separate known/accepted constraints** (e.g. in-process state under a
  single-replica assumption) from new bugs.

Resist the count. Report the honest, deduplicated set with severities you can
defend.

### Phase 5 — Verify the promoted siblings to the same standard

Sweep findings are hypotheses. Before any of them influences a decision or a
comment, run them back through the Phase-2 verifier. This is where
plausible-but-wrong findings die — the "silently skips security-critical work"
that turns out unreachable because of startup ordering. Only verified siblings
survive. Keep the wall up: a sweep gives hypotheses, only verification gives
facts.

### Phase 6 — Find and trace the contract/design question

Step back and ask: **what does this bug imply the system is supposed to
guarantee, and does the code actually decide that anywhere?** This is the
highest-leverage move in the whole process, and it is "Constraint Ownership"
from `philosophy` applied to the specific failure. Dispatch a
**contract-tracer** to gather the evidence — a `general-purpose` subagent,
since establishing what the code *guarantees* is analysis rather than search —
but make the call yourself.

- "Is `instanceId` reuse a supported contract?" → trace ID generation, the input
  validators, the lifecycle/state machine, the schema's uniqueness/FK design. If
  the schema is *hostile* to the assumed usage, the report's premise is the bug.
- "What does `ready`/`registered` guarantee, and for *what* consumer?" →
  enumerate the readers of the relevant state; if one structure serves two
  different readiness needs, the conflation *is* the defect.

The contract answer usually reframes the fix entirely, and it is often a
**product/owner decision, not a code choice.** Surface it as such rather than
silently deciding it — put the question to the user (use `AskUserQuestion` when
the answer is an enumerable set of options), rather than burying the decision in
a comment.

### Phase 7 — Stress-test the proposed fix (and your own)

Run the filed fix — and any fix *you* propose — through adversarial review. Use
two complementary adversaries, and brief each with the proposal and the evidence
behind it (there is no commit to review yet — the fix is still a proposal):

- `@greybeard` tears into the **design**: does the architecture hold, does it
  relocate the failure, does it survive version skew.
- `@critique` tears into the **claims and assumptions** — its stated job is to
  test assumptions. Point it at the load-bearing conclusions this triage rests
  on: the verifier's verdicts (is that path *really* unreachable?), the contract
  answer from Phase 6, and the assertion that the proposed fix removes rather
  than moves the failure. Tell it to argue the opposite and find where the
  reasoning breaks.

Do not mark your own homework; the review that catches the most is the one aimed
at *your* conclusions, not the report's. Check specifically:

- Does it *remove* the failure or *relocate* it? Trace the second run, the
  retry, the next step.
- Does it answer the Phase-6 contract question correctly?
- **Back-compat / version skew:** during a rolling deploy, can old talk to new
  and new to old? A new wire frame or schema field can break the old side
  mid-deploy.
- Does it widen an adjacent latent bug (the Phase-4 "opposite direction"
  sibling)?
- Is there a *simpler* fix aimed at the actual symptom that avoids a
  protocol/schema change entirely?

### Phase 8 — Scope the outcome correctly

Split the work along its real seams:

- **One tiny correctness fix** for the observed symptom, plus its repro test.
- **One orthogonal robustness fix** for the deeper blast radius, in its own
  commit/issue, gated on any pre-checks it needs.
- Do *not* fold them into one monolith, and do not let a one-liner stand in for
  the overarching fix.

When the real problem is bigger than the symptom ticket, **lift it into its own
issue that blocks the symptom ticket.** The symptom stays as the narrow repro;
the parent carries the actual fix plan as acceptance criteria. Use `linear-create`
to file the issues with the right blocking relationship. Do the in-scope work
now; don't defer in-scope work to a vague "later" (see "Scope Discipline" in
`style`).

### Phase 9 — Communicate with calibrated confidence

When you write findings into a ticket or comment:

- **Separate verified facts from opinion**, visibly. Facts are file:line-backed;
  opinion is labeled opinion and subject to debate.
- **Correct the record** for anything the report got wrong (502 not 503, the
  trigger lives in the consumer, one usage not zero) so downstream decisions rest
  on accurate ground.
- **Mark machine-generated content as such** when an agent authored it, and say
  "verify independently."
- Don't post until the human asks, if that is the working agreement.

## Roles (when running this with subagents)

Reserve expensive reasoning for judgment; delegate legwork. Keep the wall up
between hypotheses (sweeps) and facts (verification).

| Role | Job | Stance | Suggested agent |
|---|---|---|---|
| **Verifier** | Confirm/refute report claims against source | Skeptical; file:line evidence; "exists ≠ reachable" | `general-purpose` (`Explore` to gather evidence) |
| **Class-hunter** | Find sibling instances of the disease | Exhaustive; framed by disease, not symptom | `Explore` (very thorough) |
| **Contract-tracer** | Establish what the code actually guarantees | Evidence-backed verdict, even if "undefined" | `general-purpose` |
| **Design adversary** | Tear into the proposed fix's architecture | Battle-tested; ranks risk; refuses the unsafe | `@greybeard` |
| **Claims adversary** | Attack the verdicts, contract answer, and "removes not relocates" assertion | Argue the opposite; find where the reasoning breaks | `@critique` |
| **Synthesizer (you)** | Triage, decide, scope, communicate | Judgment over relay; calibrated confidence | main agent |

## Red flags in a report (each triggers deeper checks)

- Confident specific numbers (line numbers, status codes, counts) — verify every
  one; they drift, and they are wrong more often than you would expect.
- Named functions/paths you can't find — they may be in a *different repo* (a
  vendored consumer), which changes the contract question.
- A one-line fix presented as "the correct upstream fix" — ask what it relocates
  and what contract it silently decides.
- "Systemic — zero/all usages of X" — count it yourself; "systemic" framing built
  on a wrong count is still wrong.
- A fix that requires changes in more than one layer — stop and ask which layer
  *owns* the constraint; fix it there, trust it elsewhere.
- A new wire frame / schema field / status value — check rolling-deploy version
  skew before anything else.

## The one-paragraph version

Treat the report as a hypothesis. Verify every load-bearing claim against the
code with an adversarial reader. Assume the bug is one of a class and hunt the
siblings, then verify the interesting ones to the same standard — a sweep
produces hypotheses, not facts. Find the contract question the ticket didn't ask
and trace it; the answer usually reframes the fix and is often an owner's
decision. Stress-test the proposed fix (and your own) for relocation, version
skew, and adjacent bugs — don't mark your own homework. Split the result into a
tiny correctness fix plus an orthogonal robustness fix, lifting the overarching
problem into its own blocking issue. Communicate facts and opinion separately,
correct the record, and mark machine-authored content.

## Acknowledgment

After reviewing this skill, state: "I have reviewed the bug-triage skill; the
report is a hypothesis, the code is the truth."
