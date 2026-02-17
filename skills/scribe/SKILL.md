---
name: scribe
description: Maintain product, architecture, and implementation docs — routes input, detects gaps, and interviews for completeness
---

# Scribe

Use this skill to maintain documentation across multiple documents that represent different levels of abstraction. When the user provides input, analyze it and route it to the correct document. Scribe is not a passive filing system — after recording what the user provides, it actively identifies gaps, checks cross-document consistency, and asks targeted questions to strengthen the documentation.

## Document Discovery

Before processing input, locate the documentation files:

1. Search for existing files matching `PRODUCT.md`, `ARCHITECTURE.md`, and `IMPLEMENTATION.md` (case-insensitive) in:
   - Repository root
   - `docs/` directory

2. If documents exist, use their locations. If multiple matches exist for the same type, prefer the repository root.

3. If no documents exist, use these defaults when creating new ones:
   - `PRODUCT.md` in repository root
   - `ARCHITECTURE.md` in repository root
   - `IMPLEMENTATION.md` in repository root

## Document Types

**Product** - Product-level documentation
- What we're building and why
- User-facing value propositions
- Vision and goals
- Target users and use cases
- Business justification

**Architecture** - System architecture documentation
- How the system is structured
- Components and their relationships
- Abstractions and interfaces
- Data flow and control flow
- Design decisions that are technology-agnostic

**Implementation** - Implementation documentation
- Specific technology choices
- Protocols and formats
- Libraries and frameworks
- Concrete technical details
- Configuration and deployment specifics

## Execution Steps

### Step 1: Analyze Input

Read the user's input and determine which category it falls into. Classification is based on two sources: general heuristics and project-specific signals learned from existing documents.

**General heuristics:**

*Product signals:*
- Describes user needs or problems
- Explains value or benefits
- Discusses market or competitive positioning
- Uses language like "users can", "enables", "provides value"
- Talks about goals without specifying how

*Architecture signals:*
- Describes components or modules
- Explains how parts interact
- Defines abstractions or interfaces
- Discusses system properties without naming specific technologies
- Technology-agnostic design decisions

*Implementation signals:*
- Names specific technologies, protocols, or formats
- Describes wire formats or API specifications
- Specifies configuration details
- Uses language like "uses", "built on", "implemented with"
- Concrete technical choices

**Project-specific signals:**

Read the existing documents to learn the project's vocabulary. Extract key terms, component names, and patterns that indicate document ownership. For example:
- If the architecture document discusses "the kernel" and "agents", mentions of these terms suggest architectural content
- If the implementation document discusses "SMTP" and "IMAP", mentions of email protocols suggest implementation content
- If the product document discusses "wallets" as a user-facing feature, wallet mentions in a value context suggest product content

Use these learned signals alongside general heuristics. Project-specific vocabulary takes precedence when it provides a clear signal.

### Step 2: Classify and Deepen

If the categorization is clear, proceed to update the appropriate document.

If the input is ambiguous or spans multiple categories, do not simply ask "which document?" Instead, interview the user to decompose the input into distinct claims that can each be routed precisely:

1. Explain what makes the input ambiguous — identify the product, architecture, and/or implementation aspects you see in it.
2. Ask targeted questions to separate those aspects. For example:
   - "When you say 'fast and reliable', is that a promise to users (product) or a system property you need to design for (architecture)?"
   - "You mentioned [component] — is that a user-facing concept or an internal abstraction?"
3. Route each extracted piece to its appropriate document. A single user statement may result in updates to multiple documents.

### Step 3: Update Document

Read the target document to understand its current structure and content.

Determine where in the document the new content belongs:
- Does it extend an existing section?
- Does it require a new section?
- Does it modify existing content?

Make the update, maintaining the document's existing style and structure.

After updating, assess whether the change is **significant**. A change is significant if it:
- Introduces a new concept, component, or section
- Contradicts or substantially revises existing content
- Adds a top-level capability or design decision

If the change is minor — extending an existing section with more detail, fixing wording, adding a clarification — skip Steps 4 and 5 and proceed directly to Step 6.

### Step 4: Cross-Document Consistency

*Only for significant updates.*

Read the other two documents and check whether the new content implies entries that should exist in sibling documents but don't. Common patterns to look for:

- A new architecture component with no corresponding product justification
- A new product capability with no architectural description of how it works
- An implementation detail referencing a component not described in architecture
- A product goal with no implementation approach mentioned

If gaps are found, present them as a batch of 2-4 questions. For example:

> I updated ARCHITECTURE.md with the export service. I noticed:
> 1. PRODUCT.md doesn't describe data export as a user-facing capability. Should it?
> 2. IMPLEMENTATION.md has no entry for how exports are generated. Want to add that now?

For each question the user answers, update the corresponding document before proceeding.

### Step 5: Gap Detection and Completeness

*Only for significant updates.*

Scan the updated document for weaknesses:
- Concepts referenced but not elaborated
- Sections that are thin relative to their importance
- Missing failure modes, edge cases, or constraints
- Decisions stated without rationale

Present 2-4 probing questions as a batch. Focus on non-obvious gaps — things the user might not think to document unprompted. For example:

> The export service section in ARCHITECTURE.md looks good. A few things that would strengthen it:
> 1. What happens when an export fails mid-generation? Is there retry logic or does the user need to re-trigger?
> 2. Are there size or rate limits on exports?
> 3. Who has permission to trigger exports — any user, or only certain roles?

Update the document with any answers the user provides. If the user declines to answer, move on without pressing.

### Step 6: Report

Briefly confirm what was changed and in which document. If Steps 4 or 5 resulted in additional updates, summarize those as well:

> Updated ARCHITECTURE.md: added export service component under "Data Pipeline" section.
> Also updated PRODUCT.md: added data export as a user-facing capability (from consistency check).

## Examples

These examples demonstrate how classification and the active documentation steps work. The specific terms will vary by project.

### Classification

**Input:** "Users can export their data in multiple formats"
**Classification:** Product (describes user-facing capability)

**Input:** "The export service validates permissions before generating files"
**Classification:** Architecture (describes component responsibility and interaction)

**Input:** "Exports are generated as CSV using the fast-csv library"
**Classification:** Implementation (names specific format and library)

### Depth Elicitation (Step 2)

**Input:** "Data export is fast and reliable"
**Classification:** Ambiguous — has both product and architecture aspects.

Instead of asking "which document?", interview to decompose:

> "Data export is fast and reliable" touches a few layers. Let me ask a couple of questions:
> 1. Is "fast and reliable" a promise you're making to users (product) or a design constraint the system must satisfy (architecture)?
> 2. Does "fast" have a concrete target — like sub-5-second for a typical export?

If the user answers: "It's a user-facing promise, and yes, under 5 seconds for up to 10k rows" — this produces two updates:
- **PRODUCT.md:** Data export completes in under 5 seconds for typical datasets (up to 10k rows).
- **ARCHITECTURE.md:** The export pipeline must meet a 5-second latency target for datasets up to 10k rows.

### Cross-Document Consistency (Step 4)

**Scenario:** User adds "The notification service delivers messages through email, SMS, and push" to ARCHITECTURE.md.

After updating, scribe reads the other documents and finds that PRODUCT.md has no mention of notifications as a user-facing feature. It asks:

> I added the notification service to ARCHITECTURE.md. I noticed:
> 1. PRODUCT.md doesn't mention notifications as a user-facing capability. Should I add that users receive notifications via email, SMS, and push?
> 2. IMPLEMENTATION.md doesn't specify which email/SMS/push providers are used. Want to capture those choices now?

### Gap Detection (Step 5)

**Scenario:** User adds a new "Authentication" section to ARCHITECTURE.md describing token-based auth with refresh tokens.

After updating, scribe scans the section and identifies gaps:

> The authentication section covers the core flow. A few things that would strengthen it:
> 1. What happens when a refresh token is revoked — does the user get signed out immediately or at next request?
> 2. Is there a maximum session duration regardless of refresh?
> 3. How are tokens stored on the client side?

## Error Handling

### Document does not exist

If the target document does not exist, ask the user if they want to create it:

> [DOCUMENT].md does not exist. Would you like me to create it?

### Content conflicts

If the new content contradicts existing content, flag it:

> This conflicts with existing content in [DOCUMENT].md: "[existing content]". Should I replace it or should we reconcile the difference?

### Unclear scope

If the input is too broad or vague to place in a specific document:

> I'm not sure where this belongs. Can you provide more detail or specify which document (product, architecture, or implementation)?
