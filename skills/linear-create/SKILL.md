---
name: linear-create
description: Create well-structured Linear issues, projects, project updates, or initiatives
argument-hint: "[description] [--from-doc]"
---

# Linear Create

Use this skill to create properly structured Linear artifacts. Based on the scope of work, this skill will create the appropriate combination of issues, projects, project updates, and/or initiatives.

## Prerequisites

Before doing anything else, verify the Linear MCP tools are available by checking for `mcp__linear__*` tools in your toolset. If they are not available, inform the user:

> The Linear integration is not available. Please ensure the Linear MCP server is configured and try again.

Do not proceed with any other phases until this check passes.

## Phase 1: Document Discovery

When the user provides `--from-doc` or mentions a planning document, search for scribe-managed documents:

1. Look for `PRODUCT.md`, `ARCHITECTURE.md`, `IMPLEMENTATION.md` in:
   - Repository root
   - `docs/` directory

2. If no documents are found, ask the user:
   > I couldn't find any planning documents. Do you have a document you'd like me to reference?

3. When a document is found, read it and extract:
   - Features or work items mentioned
   - Technical context and constraints
   - Scope indicators (timeline mentions, complexity signals)

Use extracted information to:
- Pre-populate issue descriptions with relevant context
- Propose appropriate artifact types based on scope
- Ask targeted follow-up questions for gaps not covered by the document

## Phase 2: Analyze Input

Determine what the user wants to create:

1. **Explicit request**: User specifies artifact type ("create an issue for...", "create a project for...", "post a project update for...")
2. **From document**: User provides `--from-doc` to extract work items from planning documents
3. **Freeform**: User describes work without specifying type

Project updates are a distinct artifact: they communicate status on an existing project to a non-technical audience and are never inferred from scope. The user must explicitly ask for one.

For freeform input, estimate the scope:

| Scope | Duration | Artifact |
|-------|----------|----------|
| Small | 1-3 days | Single issue |
| Medium | 1-2 weeks | Project |
| Large | Quarter+ | Initiative with projects |

Present your assessment to the user and confirm before proceeding.

### Do Not Populate Projects with Issues Unless Asked

Creating a project does **not** imply creating issues inside it. When the user asks for a project (or scope assessment lands on a project), create the project and its milestones only. Do not break the work into issues and file them under the project unless the user has explicitly asked for issues.

When you assess a request as project-scoped, your proposal in Phase 5 should offer the project and its milestones, without issues. If you believe the work would benefit from being broken into issues, ask the user whether they want that — do not assume it. Only create issues inside a project when the user has explicitly requested them, either up front or in response to that question.

This does not apply to initiatives: an initiative is a container for projects, and the "Initiative with projects" assessment may propose the constituent projects (still without issues inside them unless asked).

## Phase 3: Interview

If information is missing, ask targeted questions. Keep interviews brief and focused.

### For Issues

Required information:
- What problem does this solve or what value does it add?
- How will we know it's done? (acceptance criteria)

Optional:
- Are there technical constraints or dependencies?
- Which team should own this?

### For Projects

Required information:
- What is the goal/outcome of this project?
- What is the target timeframe?
- Who should lead this project?

Optional:
- What teams are involved?
- What are the key milestones?

### For Initiatives

Required information:
- What strategic objective does this serve?
- Who is the executive owner?
- What projects should be included?

### For Project Updates

Required information:
- Which project is this update for? (use `mcp__linear__list_projects` and confirm with the user if the match is not exact)
- What is the project's current health? (on track, at risk, off track, completed, paused)
- What has the project unlocked or enabled since the last update? Describe in terms of capabilities, outcomes, or things that are now possible — not lists of completed tickets.
- What's coming next, framed by user-visible impact?
- Are there any risks or blockers the audience needs to know about? Describe them by impact, not implementation.

Optional:
- Should the update be tied to a specific milestone?

Before drafting, retrieve the most recent prior update with `mcp__linear__get_status_updates` so the new one continues the narrative rather than restating prior progress.

## Phase 4: Draft Content

Create drafts following these conventions:

### Do Not Reference Local Files

Linear artifacts are read by people who do not share your working directory. Do not include local file paths, line numbers, working-tree-relative paths, or instructions like "see `src/foo.ts`" in titles, descriptions, or comments. Those references rot, are not clickable, and assume context the reader does not have.

Instead:

- Describe the behavior, module, or concept in plain language ("the authentication middleware", "the request retry logic")
- Link to permanent URLs (GitHub permalinks at a specific commit, published documentation) when a precise pointer is required
- Quote the relevant code inline if a short excerpt is needed for context

This applies equally when drafting from planning documents — extract the meaning, do not transcribe paths.

### Specs Belong With the Artifact, Not as Local Paths

If a spec, design document, or planning artifact needs to be preserved so an implementer can refer to it, store it on the Linear artifact rather than referencing the local file path. Once it lives in Linear, any reference inside an issue or project description should point to it — never to the original local file path.

There are **two distinct mechanisms**, and they attach to different things. Pick by what you are attaching it to:

| Where it needs to live | Mechanism | Tool(s) |
|------------------------|-----------|---------|
| **Project** | Linear **document** | `mcp__linear__save_document` with the `project` parameter |
| **Initiative / cycle / team** | Linear **document** | `mcp__linear__save_document` with the `initiative`, `cycle`, or `team` parameter |
| **Issue** | Linear **document** OR file attachment | `mcp__linear__save_document` with `issue`, or the file-upload flow below |

**Documents are the only way to attach a spec to a project.** The file-attachment tools (`mcp__linear__prepare_attachment_upload`, `mcp__linear__create_attachment_from_upload`, `mcp__linear__create_attachment`) **target an `issue` only** — their sole parent parameter is `issue`, so they cannot attach to a project, initiative, cycle, or team. Do not attempt to upload a file "to a project"; it will fail. Instead, create a document.

#### Attaching a spec to a project (the common case)

Use `mcp__linear__save_document` with the markdown content inline and the `project` set to the project's name, ID, or slug:

```
mcp__linear__save_document(
  title: "Authentication Design Spec",
  content: "<the full spec as markdown — literal newlines, not \\n>",
  project: "<project name, ID, or slug>"
)
```

This is the right move for any spec describing a whole project's scope or design. The same tool reparents to an `issue`, `initiative`, `cycle`, or `team` by passing the corresponding parameter instead — exactly one parent is required when creating.

#### Attaching a file to an issue (binaries, PDFs, screenshots)

When the artifact is a binary (PDF, image, archive) that belongs on a single **issue**, use the file-upload flow. Prepare, PUT the raw bytes, then finalize — one file at a time, because signed URLs expire in 60 seconds:

1. `mcp__linear__prepare_attachment_upload` with `issue`, `filename`, `contentType`, `size`
2. `PUT` the raw bytes to the returned `uploadRequest.url`, sending every header verbatim
3. `mcp__linear__create_attachment_from_upload` with the `issue` and the returned `assetUrl`

For markdown specs on an issue, prefer a document (`save_document` with `issue`) over a file attachment — it stays readable and searchable inside Linear.


### Issue Format

**Title**: Clear and actionable. Verb phrases are preferred, but sentences or noun phrases are acceptable when they provide clarity.

- Good: "Add retry logic for failed API calls"
- Good: "Fix race condition in transaction verification"
- Good: "Create market validation track for <product-name>"
- Bad: "API retry" (too vague)
- Bad: "Bug in transactions" (not actionable)

**Description**:

```
# Background

<Context and what triggered this work. Optional for simple tasks.>

# Outcome

<What success looks like, with checkboxes for specific conditions>

- [ ] <Specific, testable condition>
- [ ] <Another condition>
```

For simple tasks, you can omit `# Background` and use only `# Outcome` with checkboxes.

**Labels and Priority**:

- Set priority based on urgency and impact
- Apply labels for categorization (e.g., bug, feature, tech-debt) if the workspace uses them
- Ask the user about priority and labels if not specified

When appropriate, use subsections under `# Outcome` to organize related items:

```
# Outcome

## Questions

- [ ] What are the key takeaways?
- [ ] Which parts apply to our strategy?

## Tasks

- [ ] Document findings
- [ ] Present to team
```

### Project Format

**Name**: Outcome-focused description
- Good: "User authentication with SSO support"
- Good: "Get 10 customer leads for <product-name> through direct outreach"
- Bad: "Auth work"

**Description**: Goal, scope, and any constraints.

For validation or experiment projects, use the Hypothesis/Experiment/Steps pattern:

```
Hypothesis - <What you believe to be true>

Experiment
<Experiment name>

* <Step 1>
* <Step 2>
* <Step 3>
```

**Milestones**: Key checkpoints showing progression toward the goal. Examples:

- Completion states: "Target list ready", "Outreach completed", "Analysis complete"
- Phase labels: "MVP", "Full implementation", "Polish and launch"

### Initiative Format

**Name**: Strategic objective

**Description**: Include as much information as needed to convey the business goal and how success will be measured. If unsure what to include, prompt the user for guidance.

### Project Update Format

**Audience**: Project updates are read by non-technical stakeholders — founders, GMs, customer-facing teammates, leadership, and sometimes customers. Write for someone who cares about *what the project makes possible*, not *what work was done*.

**Style rules:**

- Lead with what is now possible, available, or unblocked because of recent progress. The reader wants to know what changed for them, not what changed in the codebase.
- Do not enumerate completed issues, PR titles, commits, or internal implementation details. "Shipped INF-204, INF-205, INF-211" is the wrong shape; "Customers can now invite teammates and assign roles without contacting support" is the right shape.
- Avoid jargon, acronyms, internal codenames, and tool-of-the-week terminology unless they are already part of the audience's vocabulary. When in doubt, spell it out in plain language.
- Frame risks and blockers by their impact on the outcome ("the launch date may slip by two weeks because we are still waiting on the vendor's API access"), not by their technical cause.
- Keep it short. A project update that takes more than a minute to read will not be read.

**Structure**:

```
## Where we are

<One or two sentences naming the current state of the project in plain terms.>

## What this unlocks

<What is now possible, available, or in motion because of recent progress. Outcome-oriented, not task-oriented.>

## What's next

<The next user-visible milestone, framed by impact rather than by the tasks required to get there.>

## Risks

<Optional. Only include if there is something the audience needs to know. Describe the impact on the outcome, not the technical cause.>
```

If a section has nothing meaningful to say in this update, omit it rather than padding it.

**Health**: Set the project health to match reality (`onTrack`, `atRisk`, `offTrack`, `complete`, or `paused`). If you would not show the chosen health to the project's sponsor with a straight face, it is the wrong health.

**Self-check before posting**: Re-read the draft and ask, "would a non-engineer who has never opened the codebase come away knowing what changed for them?" If the answer is no, rewrite it.

## Phase 5: Review and Adjust

Present the complete draft to the user. For a project-scoped request, propose the project and its milestones only — do not include an issue breakdown unless the user explicitly asked for issues (see "Do Not Populate Projects with Issues Unless Asked"):

```
I propose creating:

**Project**: Add user authentication
- Lead: <to be assigned>
- Target: <target-quarter>
- Milestones:
  1. Basic auth flow complete
  2. SSO integration complete

Would you like to adjust anything before I create this? (If you'd
also like this broken into issues under the project, let me know.)
```

Only when the user has explicitly requested issues, include them in the proposal:

```
**Issues**:
1. "Set up authentication database schema"
2. "Implement login/logout flow"
3. "Integrate SSO provider"
4. "Add session management"
   - Blocked by: #2
```

Allow the user to:
- Adjust titles or descriptions
- Change the structure (e.g., "make #3 and #4 one issue")
- Add or remove items
- Specify assignees or teams

## Phase 6: Create in Linear

### Workspace Discovery

Before creating artifacts, query the Linear workspace using the appropriate `mcp__linear__*` tools:

- **Teams**: Always query available teams (`mcp__linear__list_teams`) and ask the user which team should own issues or projects
- **Projects**: Query existing projects (`mcp__linear__list_projects`) if the user mentions adding to an existing project
- **Initiatives**: Query existing initiatives (`mcp__linear__list_initiatives`) if linking to one

Present options to the user when multiple choices exist.

### Creating Artifacts

After user approval, create artifacts using the appropriate `mcp__linear__*` tools. Only create the artifacts that were actually approved — in particular, do not create issues under a project unless the user explicitly requested them (see "Do Not Populate Projects with Issues Unless Asked"). Steps 2–4 apply only when issues are part of the approved scope:

1. **Create container first** (initiative via `mcp__linear__save_initiative`, project via `mcp__linear__save_project`)
2. **Create issues** in dependency order (`mcp__linear__save_issue`) — only if issues were approved
3. **Set blocking relationships** between issues (via `mcp__linear__save_issue` parameters)
4. **Add issues to project** (via `mcp__linear__save_issue` parameters)
5. **Add projects to initiative** (via `mcp__linear__save_project` parameters)
6. **Attach specs and design docs** — for a project, create a Linear document with `mcp__linear__save_document` and the `project` parameter (the file-attachment tools only work on issues); for an issue, use a document or the file-upload flow. See "Specs Belong With the Artifact, Not as Local Paths" above.
7. **Post project updates** (via `mcp__linear__save_status_update`, targeting the project resolved during the interview)

Report created artifacts to the user with their URLs.

## Error Handling

If Linear creation fails:

1. Report the error to the user with any details provided by the API
2. List what was successfully created before the failure (with URLs if available)
3. Do not proceed with dependent artifacts if a parent fails (e.g., don't create issues if project creation failed)
4. Ask if they want to retry or adjust the request

## Quality Reminders

- Issues should be self-contained and handoff-ready at any moment
- A single issue should take no more than 2-3 days to implement
- Use many tickets if needed for clarity; they're cheap
- Status updates in tickets reduce interruptions

If an issue looks like it will take more than 3 days, suggest breaking it down.

## Common Patterns

### Bug Report to Issue

```
User: "The login page crashes when you enter special characters"

Issue:
  Title: Fix login page crash on special character input
  Description:
    # Background

    Login page crashes when users enter special characters in the
    username or password field.

    Steps to reproduce:
    1. Navigate to /login
    2. Enter "user@test" in username
    3. Page crashes

    # Outcome

    - [ ] Special characters in username field do not cause crash
    - [ ] Special characters in password field do not cause crash
    - [ ] Input is properly sanitized before processing
```

### Feature Request to Project and Issues

This pattern applies only when the user has explicitly asked for issues as well as the project (see "Do Not Populate Projects with Issues Unless Asked"). If they asked only for a project, create the project and milestones and stop.

```
User: "We need to add dark mode to the application, broken into issues"

Project: Add dark mode theme support
  Target: 2 weeks
  Milestones:
    1. Theme infrastructure complete
    2. All components themed

Issues:
  1. "Add theme context and toggle component"
  2. "Define dark mode color palette"
  3. "Update core components for theme support"
     - Blocked by: #1, #2
  4. "Add theme persistence to user preferences"
     - Blocked by: #1
```

### Validation Project

This request does not ask for issues, so the deliverable is the project and its milestones only. The experiment steps live as checkboxes in the project description, not as separate issues.

```
User: "We need to validate if customers want our new <product-name> product"

Project: Get 10 customer leads for <product-name> through direct outreach
  Lead: <to be assigned>
  Target: 2 weeks
  Description:
    Hypothesis - Teams we know and can reach out to have a need for <product-name>.

    Experiment
    Direct Outreach

    - [ ] Create list of targets
    - [ ] Create collateral if needed
    - [ ] Create strategy for outreach including any templates
    - [ ] Execute outreach
    - [ ] Conduct customer interviews
    - [ ] Analyze results

  Milestones:
    1. Target list and collateral ready
    2. Outreach completed
    3. Customer interviews recorded
    4. Analysis complete
```

Only if the user also asks for the work to be tracked as issues, add them under the project:

```
Issues:
  1. "Create target list for <product-name> outreach"
  2. "Create outreach collateral and templates"
  3. "Execute outreach campaign"
     - Blocked by: #1, #2
  4. "Conduct and record customer interviews"
     - Blocked by: #3
  5. "Analyze results and present findings"
     - Blocked by: #4
```

### Strategic Goal to Initiative

```
User: "We need to expand our platform to support enterprise customers"

Initiative: Enterprise platform expansion
  Owner: <executive-owner>
  Target: <target-quarter>

Projects:
  1. "Multi-tenant architecture" - Isolate customer data and resources
  2. "Enterprise SSO integration" - Support SAML and OIDC providers
  3. "Admin dashboard" - Self-service management for enterprise admins
  4. "Audit logging" - Compliance-ready activity tracking
```

### Planning Document to Issues

```
User: "Create issues from our product doc" or "--from-doc"

[Skill searches for PRODUCT.md, ARCHITECTURE.md, IMPLEMENTATION.md]
[Finds PRODUCT.md with feature descriptions]

Skill: I found PRODUCT.md which describes the following features:
  - User authentication with SSO
  - Usage metrics dashboard
  - Export functionality

Based on the document, I propose:

**Project**: User authentication with SSO support
  (From PRODUCT.md: "Users need secure login with enterprise SSO...")

**Issues**:
  1. "Implement basic email/password authentication"
     # Background
     From PRODUCT.md: Users need secure login...

     # Outcome
     - [ ] Users can register with email/password
     - [ ] Users can log in and log out

  2. "Integrate SAML SSO provider"
     ...

Which features would you like me to create issues for?
```
