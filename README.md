# asg - Claude Code Plugin

Development skills for code review and coding conventions.

## Installation

### Local Development

```bash
claude --plugin-dir /path/to/this/repo
```

### From Git Repository

```bash
# In Claude Code
/plugin install https://github.com/alexanderguy/skills
```

## Skills

### `/asg:code-review`

**Invocation:** Manual only (use `/asg:code-review` to invoke)

Guidance for performing code reviews and pull request reviews. Includes:

- Scope determination using git diff
- Handling pre-existing code (exempt from review)
- Delegating to sub-agents with proper context
- Test coverage philosophy (focus on business logic, trust libraries)
- Review checklist

### `/asg:style`

**Invocation:** Auto-loaded when relevant (writing code, reviewing code, discussing patterns)

General coding conventions that apply to any language:

- Documentation guidelines (avoiding redundant comments)
- Git commit message format
- Code reuse and refactoring (search before implementing, prompt with options)
- Build verification (full build before declaring complete)
- Configuration files (don't modify unless asked)
- Personality (no emojis, act professionally)

### `/asg:typescript`

**Invocation:** Auto-loaded when relevant (writing TypeScript, reviewing TypeScript, discussing type system design)

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
