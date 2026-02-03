# alexanderguy-skills - Claude Code Plugin

Development skills for code review and coding conventions.

## Installation

### Local Development

```bash
claude --plugin-dir /path/to/this/repo
```

### From Git Repository

```bash
# In Claude Code
/plugin marketplace add alexanderguy/agent-marketplace
/plugin install alexanderguy-skills@agent-plugins
```

## Skills

### `linear-issue-workflow`

Structured workflow for implementing features or fixes tracked in Linear. Includes:

- Fetching issue details and acceptance criteria
- Creating implementation plans with user approval
- Setting up git worktrees for isolated work
- Self-review before pushing
- PR creation with proper Linear issue linking

### `pull-request-review`

Review a pull request by branch name or URL. This skill:

- Accepts a branch name or GitHub/GitLab PR/MR URL
- Creates a git worktree at `../worktree/<branch-name>` relative to repo root
- Loads the `code-review` skill to perform the actual review
- Keeps the worktree available for further investigation after review

### `code-review`

Guidance for performing code reviews and pull request reviews. Includes:

- Scope determination using git diff
- Handling pre-existing code (exempt from review)
- Delegating to sub-agents with proper context
- Test coverage philosophy (focus on business logic, trust libraries)
- Review checklist

### `style`

General coding conventions that apply to any language:

- Documentation guidelines (avoiding redundant comments)
- Git commit message format
- Code reuse and refactoring (search before implementing, prompt with options)
- Build verification (full build before declaring complete)
- Configuration files (don't modify unless asked)
- Personality (no emojis, act professionally)

### `typescript`

TypeScript-specific conventions and type system patterns:

- Naming conventions (files, types, functions, variables, acronyms)
- Type system patterns (runtime validation, type guards, generics)
- Async patterns (factory functions, parallel execution, timeouts, retries)
- Import/export patterns (barrel exports, named exports, import ordering)
- Error handling idioms
- Testing patterns
- TSDoc documentation

## Customization

These skills provide general guidance. For project-specific conventions:

1. Create a project-level `CLAUDE.md` or `.claude/settings.json` with overrides
2. Reference specific conventions that differ from these defaults
3. The project-level instructions take precedence

## License

MIT
