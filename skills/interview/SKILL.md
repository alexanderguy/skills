---
name: interview
argument-hint: "<topic>[; <context: prior decisions, file paths, constraints>]"
description: Conduct an iterative multiple-choice interview to flesh out ill-defined requirements. Uses AskUserQuestion repeatedly until obvious gaps are filled, then returns a structured summary in-conversation. Use for complex product/feature discovery where requirements are vague.
tools:
  - AskUserQuestion
---

# Interview

Use this skill to flesh out an ill-defined topic by asking the user **multiple-choice questions** in batches, iterating until no obvious gaps remain. The skill **does not write files** and **does not invoke other skills or agents** — it returns a structured summary in the conversation and stops. The caller decides what to do with the summary.

## When to use

- Complex product or feature request with vague scope ("build a notification system", "add a dashboard")
- Multiple dimensions need exploration (UX, behavior, edge cases, priorities, failure modes)
- The caller needs concrete user-facing requirements before they can plan or build

Do **not** use this skill for:
- Directive requests where the user has already decided ("create a package that does X with Y") — just plan it
- Simple A-or-B preferences — call `AskUserQuestion` directly
- Technical or architectural decisions (see §5 for the boundary test) — those don't belong in an interview at all

## Argument

The skill expects: `<topic>[; <context>]` (semicolon-separated)

- **Topic**: what the interview is about (e.g., "notification system requirements")
- **Context** (optional but recommended): what's already known — prior decisions, file paths, constraints, summaries of related code

If no argument is provided, ask the user what the topic is before proceeding.

### Using caller-provided context

If the caller passed context, **treat each fact in it as an already-answered dimension**. Do not re-ask. Examples:

- Context says "must integrate with Postgres" → don't ask about data store options
- Context references a spec or prior conversation → read it; only ask about dimensions it doesn't cover
- Context says "for internal users only" → don't ask about public-facing concerns

When in doubt about whether context settles a dimension, ask one confirmation question rather than re-litigating it.

## Process

### 1. Establish dimensions

Before asking anything, internally enumerate the dimensions worth probing. **Always probe in this order** (some dimensions can be skipped, but never reorder):

1. **Objective** — what does success look like? What's explicitly out of scope?
2. **Priorities & trade-offs** — what matters most? What's negotiable? (Asking this early steers every later question.)
3. **Users / actors** — who interacts with this? What are their roles?
4. **Functional requirements** — what must it do? Features, behaviors, capabilities.
5. **UX / interaction model** — surfaces, flows, accessibility, defaults.
6. **Data model** — what entities, what shape, what lifecycle, what retention.
7. **Integration points** — APIs, databases, external services, existing modules.
8. **Non-functional requirements** — performance, scale, latency, reliability, security, compliance.
9. **Edge cases & failure modes** — what can go wrong? What's the desired behavior when it does?
10. **Constraints** — technical limits, business rules, deadlines, team capacity.

Not every dimension applies. Prune ruthlessly. Add domain-specific dimensions where relevant, but always keep #1 and #2 first.

### 2. Ask in batches

Each round uses `AskUserQuestion`. Refer to the tool's own documentation for parameter limits, multi-select behavior, and the auto-added "Other" option — don't duplicate or work around them.

**Quality bar for options:**

- Options must be mutually exclusive and concrete — not "yes / no / maybe", not vague labels
- Each option should reflect a real, defensible choice — not a strawman
- Descriptions should make trade-offs visible ("simpler but less flexible", "consistent with existing patterns", "more work now, less later")
- Ground options in context the user has given or the codebase you've seen — don't invent generic options when concrete ones exist
- Use multi-select only when the dimension genuinely permits multiple answers (e.g., "which channels do you want?"). Default is single-select.
- If you have a recommendation, put it first and signal it in the label.

**Batching policy (resolves the tension between adaptivity and throughput):**

- **Default: 2–4 questions per round.** Bundle dimensions whose questions are reasonable regardless of how the others get answered.
- **Drop to 1 question** only when the next dimension's *question text or options* genuinely cannot be authored without first knowing this answer.
- "I might phrase the description slightly differently" is **not** a reason to split — phrase it generically and move on.
- Referencing a prior answer inside a later question is fine and encouraged ("Given you chose email-only, should retries...?"). Referencing ≠ restating the whole conversation.

### 3. Decide when to stop

After each round, judge whether any **obvious** gaps remain. A gap is obvious if:

- A reasonable engineer planning this work would still have to guess
- Skipping it would force the next step (planning or implementation) to make a load-bearing assumption
- The answer materially changes what gets built

**Stop and emit the summary when any of these triggers:**

- **Coverage**: every prioritized dimension is either answered or explicitly out of scope.
- **Diminishing returns**: remaining unknowns are minor implementation details that whoever builds this can reasonably decide.
- **Soft round cap**: you have completed **5 rounds**. If you genuinely believe more is needed, surface the remaining gaps in the summary under "Open questions deferred" and stop anyway — let the caller request a continuation.
- **User fatigue**: the user has, across the interview, hit ≥3 of any of these signals: declined to choose ("just pick something", "you decide"), answered "Other" with a short non-substantive reply, or asked you to wrap up.
- **Scope explosion**: if user answers keep widening the topic faster than they narrow it, stop and summarize what's known — see §4.

Do **not** keep asking just to feel thorough. Diminishing returns are real.

### 4. Handling trouble

**Contradiction with a prior answer.** If a new answer contradicts an earlier one, ask one clarifying question that surfaces the contradiction directly with the two earlier choices as options ("Earlier you indicated A, now B — which is correct?"). Do not silently overwrite. Record the resolved answer.

**An "Other" answer that invalidates the dimension list.** If the user's freeform reply reveals a dimension you hadn't enumerated, add it to the dimension list and continue. If it reveals the topic itself is different from what you started with, stop, summarize what's known, and tell the caller the topic has shifted.

**Bail out entirely** (stop without a full summary; report the situation to the caller) when:

- The request reduces to a technical/architectural decision the user cannot reasonably make (see §5). Tell the caller a technical decision is required before further user input is useful.
- The user is fundamentally undecided about the objective itself, not just details — there is nothing to interview about yet.
- Scope keeps expanding round after round with no anchor. Report scope ambiguity and let the caller decide whether to split the topic.

In a bail-out, emit a brief note: what you learned, why you stopped, what the caller should consider doing next. Do not pretend the summary is complete.

### 5. What belongs in this interview

The test for whether a question belongs in this interview:

> **Does the answer primarily determine *what the product does* (UX, behavior, scope, priorities), or *how it is built* (architecture, libraries, data structures, protocols)?**

- *What it does* → interview question. Ask it.
- *How it's built* → **not** an interview question. The user is not the right source for this decision; it belongs in a separate technical discussion the caller can arrange.
- Both → interview the user about the user-visible aspect only. The implementation choice falls out from there or is decided elsewhere.

Examples:

- "Should notifications be email, in-app, or both?" → **interview** (user-visible behavior).
- "Should the notification service use a queue or direct delivery?" → **not interview** (implementation detail invisible to the user).
- "Auth: JWT or sessions?" → **not interview** (implementation choice), unless the user has a stated preference that constrains it.
- "Should users stay logged in across browser restarts?" → **interview** (user-visible behavior); the underlying mechanism is decided elsewhere.

If a question's answer doesn't change anything user-visible, it doesn't belong in this interview.

### 6. Return a structured summary

When the interview is done, emit the summary **inline in the conversation** (no file write). Pick the form that matches the depth of the interview:

**Compact form** (use when fewer than 3 dimensions were meaningfully probed, or the topic is small):

```
## Interview summary: <topic>

**Objective:** <one or two sentences>

**Requirements:**
- <bullet>
- <bullet>

**Open questions deferred:** <none, or list with reasons>
```

**Full form** (use for substantive interviews touching multiple dimensions). Omit any section that genuinely doesn't apply — do not pad:

```
## Interview summary: <topic>

### Objective
<one or two sentences>

### Priorities & trade-offs
<what matters most, what's negotiable>

### Requirements
- <requirement 1>
- <requirement 2>

### UX / interaction
- <flows, defaults, surfaces>

### Data & integration
- <entities, external systems>

### Non-functional requirements
- <perf / scale / reliability / security>

### Edge cases & failure handling
- <what can go wrong and the chosen behavior>

### Constraints
- <hard limits, deadlines, must-haves>

### Open questions deferred
- <anything the user punted, with the reason>
```

After emitting the summary, **stop**. Do not load other skills. Do not invoke other agents. Do not start follow-on work. Return control to the caller.

## Worked example

**Invocation:** `skill(name="interview", arguments="notification system; backend is Node/Postgres, internal users only, must integrate with existing auth")`

**Round 1** (3 questions, bundled because none depends on the others):

```
AskUserQuestion([
  { header: "Goal", question: "What is the primary goal of the notification system?",
    options: [
      { label: "Alert on critical events", description: "Errors, security issues, SLA breaches" },
      { label: "Keep users informed of activity", description: "Mentions, replies, updates" },
      { label: "Drive user re-engagement", description: "Digests, reminders, summaries" } ] },
  { header: "Priorities", question: "If you had to pick one, which matters most?",
    options: [
      { label: "Reliability of delivery", description: "Never miss a notification, even if delayed" },
      { label: "Latency", description: "Real-time, even if some are dropped under load" },
      { label: "User control", description: "Fine-grained per-event opt-in/out" } ] },
  { header: "Channels", question: "Which delivery channels do you want?", multiSelect: true,
    options: [
      { label: "In-app", description: "Notification center in the UI" },
      { label: "Email", description: "Per-event or digest" },
      { label: "Webhook", description: "Outbound HTTP to a user-configured endpoint" } ] }
])
```

**Hypothetical answers:** Alert on critical events; Reliability of delivery; In-app + Email.

**Round 2** (2 questions, now shaped by round 1):

```
AskUserQuestion([
  { header: "Email cadence", question: "Given you chose reliable delivery on email, how should emails be sent?",
    options: [
      { label: "Immediately per event", description: "Every event triggers an email" },
      { label: "Batched every 15 min", description: "Bundle bursts to reduce inbox noise" },
      { label: "Daily digest", description: "One email per day summarizing events" } ] },
  { header: "Failure", question: "If a delivery attempt fails, what's the desired behavior?",
    options: [
      { label: "Retry with backoff up to 1h", description: "Best-effort, then drop" },
      { label: "Retry until delivered", description: "Persist in queue indefinitely" },
      { label: "Retry 3x then alert ops", description: "Surface persistent failures" } ] }
])
```

**Round 3** would probe edge cases (rate limiting, dedup). After that, dimensions are covered — emit the **full-form** summary and stop.

## Style

- **Don't lecture between rounds.** A short orientation sentence ("Now let's pin down failure handling.") is fine. Avoid monologues.
- **Don't summarize the user's answers back at them.** Tracking is silent; the consolidated view appears only in the final summary. Referencing a prior answer *inside* a new question's text is fine ("Given you chose email-only…").
- **Don't ask leading questions.** "Should we use the better approach?" is bad. Present options on their merits.
- **Don't ask the user to make technical decisions they don't have context for.** Apply the §5 test.

## Anti-patterns

- **Interviewing yourself.** This skill is for getting answers from the user. If you find yourself filling in answers because they "seem obvious", stop and either ask or note them as assumptions in the summary.
- **One question per round, ten rounds deep.** Batch related questions (see §2 batching policy). Ten single-question rounds is a worse experience than three well-composed batches.
- **Asking about everything.** Some dimensions don't apply. Prune ruthlessly.
- **Treating "Other" as a failure.** When the user types a custom answer, that's signal — incorporate it and let it reshape later rounds.
- **Forgetting context.** If the caller passed context, read it. Don't ask the user things the context already answers.
- **Writing files.** This skill never writes a file — no spec, no `tmp/`, nothing. The summary is conversational. If the caller wants persistence, that's their job.
- **Invoking other skills or agents.** Do not load any other skill or hand off to any other agent from inside the interview. Emit the summary and stop.
- **Asking architecture questions in user language.** "JWT or sessions?" is not a real interview question (see §5). Reframe to user-visible behavior, or note it as outside this interview's scope.
- **Running past the soft cap silently.** If you hit 5 rounds, stop — surface remaining gaps in the summary rather than asking a 6th round.
