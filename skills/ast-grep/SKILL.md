---
name: ast-grep
description: Bulk code refactoring using AST patterns instead of manual read-edit-write cycles. Load this skill when renaming, changing signatures, or migrating API usage across many files.
---

# ast-grep

Use `ast-grep` (CLI: `sg`) for structural code search and rewriting. It matches and transforms code using Abstract Syntax Tree patterns rather than text, so it understands code structure and handles formatting, whitespace, and nesting correctly.

Prefer ast-grep over manual read-edit-write cycles when:

- Renaming functions, methods, types, or variables across files
- Changing function signatures (adding/removing/reordering arguments)
- Migrating API calls from one shape to another
- Updating import paths or restructuring imports
- Applying the same structural transformation to many call sites
- Any change where the pattern is "find all X shaped like this, replace with Y"

Do not use ast-grep when:

- The change is to a single location and you already know exactly where it is
- The transformation depends on runtime semantics ast-grep cannot see (e.g., type resolution across modules)
- The target language is not supported

## Supported Languages

JavaScript, TypeScript, TSX, JSX, Python, Rust, Go, Java, C, C++, C#, Kotlin, Swift, Scala, Ruby, PHP, Lua, Bash, Dart, Elixir, Haskell, HTML, CSS, JSON, YAML.

Not supported: SCSS, Vue, Svelte, OCaml, SQL, Dockerfile, XML, TOML.

## Pattern Syntax

Patterns are code snippets in the target language with metavariable placeholders.

### Metavariables

| Syntax | Meaning |
|---|---|
| `$NAME` | Matches exactly one AST node, captured as `NAME` |
| `$_` | Matches one node, not captured |
| `$$$NAME` | Matches zero or more sibling nodes, captured as `NAME` |
| `$$$` | Matches zero or more siblings, not captured |

**Same-name constraint:** Two occurrences of the same metavariable in one pattern must match identical text. `foo($X, $X)` matches `foo(a, a)` but not `foo(a, b)`.

### Pattern Rules

1. A pattern must parse as a single AST node (or sibling sequence with `$$$`).
2. Multi-statement patterns are not supported. `x = 1; y = 2` will error. Use relational rules (`inside`, `has`, `follows`) for multi-node relationships.
3. Optional syntax (like `extends` on a class) must appear in the pattern to match nodes that have it. Use `$$$` to absorb optional parts: `class $NAME $$$REST { $$$BODY }` matches both `class Foo {}` and `class Foo extends Bar {}`.
4. Code inside comments is not matched by patterns (a commented-out `console.log(...)` will not match pattern `console.log($$$)`). However, comment nodes themselves can be targeted using `kind: comment` in YAML rules.
5. Whitespace and formatting differences are ignored — the match is structural, not textual.

## Inline Patterns with `sg run`

Use `sg run` for quick, one-off search and rewrite operations. This is the default approach — reach for YAML rules only when you need constraints, transforms, or relational matching.

### Search

```bash
sg run --pattern 'console.log($$$ARGS)' --lang js src/
```

### Search and rewrite (preview diff)

```bash
sg run --pattern 'console.log($$$ARGS)' --rewrite 'logger.info($$$ARGS)' --lang js src/
```

### Apply rewrites in place

```bash
sg run --pattern 'console.log($$$ARGS)' --rewrite 'logger.info($$$ARGS)' --lang js -U src/
```

The `-U` (`--update-all`) flag applies changes to files without prompting. Without it, ast-grep prints a diff preview.

### Key flags

| Flag | Purpose |
|---|---|
| `-p, --pattern` | AST pattern to match |
| `-r, --rewrite` | Replacement template using captured metavariables |
| `-l, --lang` | Target language |
| `-U, --update-all` | Apply rewrites in place |
| `--globs` | Filter files by glob (prefix `!` to exclude) |
| `--json` | Structured JSON output |
| `--debug-query=<mode>` | Show AST structure; modes: `pattern` (pattern parse tree), `ast` (named nodes), `cst` (full tree), `sexp` (S-expression). Requires `--lang`. |

### Common inline recipes

**Rename a function:**
```bash
sg run -p 'oldName($$$ARGS)' -r 'newName($$$ARGS)' -l js -U src/
```

**Change an import source:**
```bash
sg run -p 'import $$$ITEMS from "old-package"' -r 'import $$$ITEMS from "new-package"' -l ts -U src/
```
Use `$$$ITEMS` (not `$ITEMS`) because `import type` inserts an extra `type` node as a sibling before the import clause. `$ITEMS` expects exactly one node in that position and fails when two are present.

**Add an argument to a call:**
```bash
sg run -p 'client.get($URL)' -r 'client.get($URL, { timeout: 5000 })' -l ts -U src/
```

**Wrap a call with an additional outer call:**
```bash
sg run -p 'fetchData($$$ARGS)' -r 'withRetry(() => fetchData($$$ARGS))' -l ts -U src/
```

**Unwrap a wrapper (Rust):**
```bash
sg run -p '$EXPR.unwrap()' -r '$EXPR?' -l rust -U src/
```

**Remove a function call, keep the argument:**
```bash
sg run -p 'deprecated($VALUE)' -r '$VALUE' -l js -U src/
```

## YAML Rules

Use YAML rules when you need constraints, transforms, relational matching, or complex multi-part logic that inline patterns cannot express.

### Basic rule structure

```yaml
id: replace-console-log
language: javascript
rule:
  pattern: console.log($$$ARGS)
fix: logger.info($$$ARGS)
```

Run a single rule file:
```bash
sg scan --rule my-rule.yaml src/
sg scan --rule my-rule.yaml -U src/
```

### Inline YAML rules

For quick one-offs that need rule features but not a file:
```bash
sg scan --inline-rules '
id: example
language: javascript
rule:
  pattern: console.log($$$ARGS)
fix: logger.info($$$ARGS)
' src/
```

### Constraints

Filter metavariable matches by node kind or regex:

```yaml
id: ban-untyped-empty-objects
language: typescript
rule:
  pattern: "const $NAME: {} = $VALUE"
constraints:
  NAME:
    regex: "^[a-z]"
message: "Avoid empty object type {}; use Record<string, unknown> instead"
```

Constraints narrow which matches a rule reports or rewrites. Use `regex` to filter by the matched text content, or `kind` to filter by AST node type.

### Relational rules

Match based on the structural position of nodes relative to each other:

```yaml
rule:
  pattern: console.log($$$)
  inside:
    kind: function_declaration
    stopBy: end
```

**`stopBy: end` is critical.** Without it, `inside` only checks one level up in the tree. With `stopBy: end`, it traverses all ancestors up to the file root. This matters because the AST has intermediate nodes you may not expect (e.g., `expression_statement` between a function call and the enclosing function body). Always add `stopBy: end` unless you specifically want single-parent-only matching.

Available relational rules:

| Rule | Meaning |
|---|---|
| `inside` | Node is a descendant of a matching ancestor |
| `has` | Node has a descendant matching this |
| `follows` | Node is preceded by a matching sibling |
| `precedes` | Node is followed by a matching sibling |

All accept `stopBy` with three valid forms: `neighbor` (only check adjacent — the default when omitted), `end` (traverse all the way to the root), or a rule object (e.g., `stopBy: { kind: function_declaration }` to stop at a specific node type).

### Matching by node kind

Use `kind` to match all AST nodes of a given type regardless of content:

```yaml
id: find-arrow-functions
language: typescript
rule:
  kind: arrow_function
```

This matches every arrow function in the codebase. Combine with `has`, `inside`, or `constraints` to narrow further. Use `--debug-query=ast` on a representative code snippet to discover the node kind names for your target language.

### Combinators

Compose rules with boolean logic:

```yaml
rule:
  all:
    - pattern: $FUNC($$$ARGS)
    - not:
        pattern: logger.$_($$$)
```

| Combinator | Meaning |
|---|---|
| `all` | All sub-rules must match (AND) |
| `any` | Any sub-rule must match (OR) |
| `not` | Sub-rule must not match (NOT) |

### Transforms

Derive new metavariables for use in `fix`:

```yaml
id: snake-to-camel
language: typescript
rule:
  pattern: $FUNC($$$ARGS)
transform:
  CAMEL_NAME:
    convert:
      source: $FUNC
      toCase: camelCase
fix: $CAMEL_NAME($$$ARGS)
```

Available transforms:

| Transform | Purpose |
|---|---|
| `convert` | Change case (`upperCase`, `lowerCase`, `camelCase`, `snakeCase`, `pascalCase`, `kebabCase`) |
| `substring` | Extract a substring by char index |
| `replace` | String find-and-replace within a metavar |
| `rewrite` | Apply sub-rewriters to a metavar (for nested transformations) |

## Debugging Non-Matching Patterns

When a pattern does not match what you expect:

1. **Inspect your pattern's AST.** Use `--debug-query=pattern` to see how ast-grep parses your pattern:
   ```bash
   sg run --pattern 'your_pattern($X)' --lang js --debug-query=pattern
   ```

2. **Inspect the source code's AST.** Use the target code itself as the pattern to see its tree structure:
   ```bash
   sg run --pattern 'myFunc(arg1, arg2)' --lang js --debug-query=ast
   ```
   This shows you the node kinds in the source, which tells you what your real pattern needs to match against. Compare the AST of your pattern (step 1) with the AST of the source to find the mismatch.

3. **Common causes of non-matches:**
   - Optional syntax missing from pattern (add `$$$` to absorb it)
   - Multi-statement pattern (not supported — use relational rules)
   - Wrong node granularity (pattern matches expression but code is in a statement context, or vice versa)
   - Language alias mismatch (`ts` vs `typescript` vs `tsx` — use the right one for the file type)

## Exit Codes

`sg run`: exit `0` means matches were found, exit `1` means no matches. This is inverted from what most tools do. Other exit codes indicate errors (e.g., 3 for missing required `--lang`, 8 for patterns that fail to parse such as multi-statement patterns). Do not treat all non-zero exits as "no matches." Note that some malformed patterns may still exit 0 with a warning about an `ERROR` node — always check stderr output.

`sg scan`: exit `0` means no error-severity findings, exit `1` means at least one `severity: error` finding. All other severities (`warning`, `info`, `hint`) exit `0`.

## Language Detection

When running on file paths, ast-grep auto-detects the language from file extensions. The `--lang` flag is required when using `--stdin` (no file extension to infer from) or when using `--debug-query` (always requires explicit language). You can omit `--lang` for normal search and rewrite operations targeting directories or specific files.

## Workflow Guidance

### Before applying rewrites

1. **Preview first.** Run without `-U` to see the diff. Review it.
2. **Scope narrowly.** Use `--globs` or specific directory paths to limit the blast radius.
3. **Verify after.** Run the project's build and test suite after applying changes to catch anything ast-grep's structural matching could not anticipate (e.g., semantic changes that happen to share syntax with the pattern).

### Choosing inline vs YAML

| Situation | Use |
|---|---|
| Simple rename or argument change | `sg run -p ... -r ...` |
| Need to exclude certain matches | YAML rule with `not` or `constraints` |
| Need positional context (inside a function, after an import) | YAML rule with `inside`/`follows`/`precedes` |
| Need case conversion or string manipulation in the replacement | YAML rule with `transform` |
| Applying multiple related transformations | Multiple `sg run` commands in sequence, or multiple YAML rules |

### Combining with manual edits

ast-grep handles the bulk structural transformation. Use manual edits for:

- New code that has no existing pattern to transform from
- Changes that require understanding type relationships across files
- One-off fixups after a bulk rewrite (e.g., adjusting a special case that the pattern caught incorrectly)

The ideal workflow for a large refactor: ast-grep for the mechanical bulk, manual edits for the exceptions, build verification to confirm everything holds together.

## Acknowledgment

After reviewing this skill, state: "I have reviewed the ast-grep skill."
