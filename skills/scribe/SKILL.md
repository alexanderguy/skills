---
name: scribe
description: Route user input to the correct document (product, architecture, or implementation) based on content analysis
---

# Scribe

Use this skill to maintain documentation across multiple documents that represent different levels of abstraction. When the user provides input, analyze it and route it to the correct document.

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

### Step 2: Confirm or Ask

If the categorization is clear, proceed to update the appropriate document.

If the input is ambiguous or could belong to multiple documents, ask the user:

> This could be [product/architecture/implementation]. Which document should I update?

Provide brief reasoning for why it's ambiguous.

### Step 3: Update Document

Read the target document to understand its current structure and content.

Determine where in the document the new content belongs:
- Does it extend an existing section?
- Does it require a new section?
- Does it modify existing content?

Make the update, maintaining the document's existing style and structure.

### Step 4: Report

After updating, briefly confirm what was changed and in which document.

## Examples

These examples demonstrate how classification works. The specific terms will vary by project.

**Input:** "Users can export their data in multiple formats"
**Classification:** Product (describes user-facing capability)

**Input:** "The export service validates permissions before generating files"
**Classification:** Architecture (describes component responsibility and interaction)

**Input:** "Exports are generated as CSV using the fast-csv library"
**Classification:** Implementation (names specific format and library)

**Input:** "Data export is fast and reliable"
**Classification:** Ambiguous - could be product (user-facing promise) or architecture (system property). Ask the user.

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
