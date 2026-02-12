---
name: linear-create
description: Create well-structured Linear issues, projects, or initiatives
argument-hint: "<description or 'breakdown: <feature>'>"
---

# Linear Create

Use this skill to create properly structured Linear artifacts. Based on the scope of work, this skill will create the appropriate combination of issues, projects, and/or initiatives.

## Phase 1: Analyze Input

Determine what the user wants to create:

1. **Explicit request**: User specifies artifact type ("create an issue for...", "create a project for...")
2. **Breakdown request**: User says "breakdown: <feature>" or describes something large that needs decomposition
3. **Freeform**: User describes work without specifying type

For freeform input, estimate the scope:

| Scope | Duration | Artifact |
|-------|----------|----------|
| Small | 1-3 days | Single issue |
| Medium | 1-2 weeks | Project with issues |
| Large | Quarter+ | Initiative with projects |

Present your assessment to the user and confirm before proceeding.

## Phase 2: Interview

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

## Phase 3: Draft Content

Create drafts following these conventions:

### Issue Format

**Title**: Clear and actionable. Verb phrases are preferred, but sentences or noun phrases are acceptable when they provide clarity.

- Good: "Add retry logic for failed API calls"
- Good: "Fix race condition in transaction verification"
- Good: "Create market validation track for Interchange"
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
- Good: "Get 10 Customer Leads for Interchange Through Direct BD"
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

## Phase 4: Review and Adjust

Present the complete draft to the user:

```
I propose creating:

**Project**: Add user authentication
- Lead: <to be assigned>
- Target: Q2 2024
- Milestones:
  1. Basic auth flow complete
  2. SSO integration complete

**Issues**:
1. "Set up authentication database schema"
2. "Implement login/logout flow"
3. "Integrate SSO provider"
4. "Add session management"
   - Blocked by: #2

Would you like to adjust anything before I create these?
```

Allow the user to:
- Adjust titles or descriptions
- Change the structure (e.g., "make #3 and #4 one issue")
- Add or remove items
- Specify assignees or teams

## Phase 5: Create in Linear

After user approval, create artifacts using the `linear` subagent:

1. **Create container first** (initiative or project)
2. **Create issues** in dependency order
3. **Set blocking relationships** between issues
4. **Add issues to project** (if applicable)
5. **Add projects to initiative** (if applicable)

Example subagent calls:

```
Task(subagent_type="linear", prompt="Create a project titled 'User authentication with SSO support' with target date Q2 2024, description: '<description>'")

Task(subagent_type="linear", prompt="Create an issue titled 'Set up authentication database schema' with description: '<desc>' and acceptance criteria: '<ac>', add to project <project-id>")

Task(subagent_type="linear", prompt="Set issue <issue-B-id> as blocked by <issue-A-id>")
```

Report created artifacts to the user with their URLs.

## Quality Reminders

From the philosophy skill:
- Issues should be self-contained (handoff-ready at any moment)
- Single issue = 2-3 days max implementation time
- Use many tickets if needed for clarity
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

```
User: "We need to add dark mode to the application"

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

```
User: "We need to validate if customers want our new Interchange product"

Project: Get 10 Customer Leads for Interchange Through Direct BD
  Lead: <to be assigned>
  Target: 2 weeks
  Description:
    Hypothesis - Teams we know and can reach out to have a need for Interchange.

    Experiment
    Direct Outreach

    * Create list of targets
    * Create collateral if needed
    * Create strategy for outreach including any templates
    * Execute outreach
    * Conduct customer interviews
    * Analyze results

  Milestones:
    1. Target list and collateral ready
    2. Outreach completed
    3. Customer interviews recorded
    4. Analysis complete

Issues:
  1. "Create target list for Interchange outreach"
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
  Owner: <VP Engineering>
  Target: Q3 2024

Projects:
  1. "Multi-tenant architecture" - Isolate customer data and resources
  2. "Enterprise SSO integration" - Support SAML and OIDC providers
  3. "Admin dashboard" - Self-service management for enterprise admins
  4. "Audit logging" - Compliance-ready activity tracking
```
