---
name: code-crafter
version: 2.4.1
description: Intelligent code refactoring, formatting, and style enforcement with AST-aware transformations
author: SMOUJBOT (OpenClaw Engine)
tags:
  - refactoring
  - style
  - formatting
  - cleanup
  - quality
dependencies:
  - black>=23.0.0
  - isort>=5.12.0
  - autoflake>=2.0.0
  - ruff>=0.0.270
  - prettier>=3.0.0
  - clang-format>=15.0.0
  - gofmt (system)
  - rustfmt (system)
language_priority:
  - python
  - typescript
  - javascript
  - cpp
  - go
  - rust
  - proto
minimum_openclaw_version: "1.8.0"
---

# Code Crafter

AST-powered code refactoring, formatting, and style enforcement engine.

## Purpose

Real use cases:
- Clean up legacy code with mixed formatting before merging PR
- Enforce team style guide (Google, PEP8, Airbnb) across monorepo
- Update deprecated APIs safely across codebase
- Extract repeated code blocks into reusable functions
- Simplify complex conditionals and nested expressions
- Remove unused imports and dead code automatically
- Convert ES5 to ES6+ patterns (arrow functions, const/let, template literals)
- Standardize string quoting, trailing commas, and line endings
- Prepare code for technical debt review by reducing cyclomatic complexity

## Scope

Commands execute in current working directory unless path specified.

### Formatting commands

```
$ code-crafter format <path>
--language <py|ts|js|cpp|go|rs|proto>
--style <pep8|google|airbnb|llvm|gofmt|rustfmt>
--line-length <int>
--dry-run (preview only)
--check (exit 1 if changes needed)
--exclude <pattern> (glob, repeatable)
--backend <local|github-actions>
$ code-crafter format-all [--staged] [--since <branch>]
$ code-crafter fix-imports <path> --profile <isort-profile>
$ code-crafter lint <path> --fix [--ignore <codes>]
```

### Refactoring commands

```
$ code-crafter extract-function <file> --region <start>:<end> --name <new_name>
$ code-crafter inline-variable <file> --variable <name>
$ code-crafter rename-symbol <file> --old <name> --new <new_name> [--scope <local|global>]
$ code-crafter simplify-conditionals <file> --target <line>|<region>
$ code-crafter expand-math <file> --target <line>|<region>
$ code-crafter convert-to-arrow <file> --function <name>
```

### Analysis commands

```
$ code-crafter complexity <path> --threshold <n> [--format <text|json|csv>]
$ code-crafter duplicate-code <path> --min-lines <n> --min-similarity <0.0-1.0>
$ code-crafter dead-code <path> --confidence <high|medium|low>
```

### Project-wide operations

```
$ code-crafter apply-rule <rule_file> --path <path>
$ code-crafter batch <operations_file.yaml> --report <report.json>
```

## Work Process

### 1. Preparation
- Parse `pyproject.toml`, `.prettierrc`, `.clang-format`, etc. for project-wide config
- Load language-specific parsers based on file extension
- Create backup snapshot: `cp <file> <file>.codecrafter.bak.<timestamp>`
- Detect modified files via `git diff --name-only` when `--staged` used

### 2. Formatting phase (idempotent)
- Run Black/Prettier with detected style options
- Apply isort/rustfmt after main formatter to avoid conflicts
- Verify syntax with language compiler (python -m py_compile, tsc --noEmit, etc.)
- Check for formatting changes; if none, skip commit step

### 3. Refactoring phase (destructive, requires review)
- Apply AST transformations on original source
- Preserve comments and docstrings unless explicitly removed
- Generate unified diff for manual approval if not `--auto`
- Validate: no syntax errors, no broken imports

### 4. Verification
- Run project test suite if present: `pytest`, `npm test`, `go test`, `cargo test`
- Build check: `python -m build`, `npm run build`, `go build ./...`
- Lint final result with `ruff check`, `eslint`, `golangci-lint`
- Fail if any verification step fails unless `--force` used

### 5. Commit (optional)
- Stage all changed files: `git add -u`
- Commit message template:
  ```
  refactor: [Brief description]

  - Auto-formatted with code-crafter
  - Applied X refactorings
  - All tests passing

  Verified: <timestamp>
  ```
- Push if `--push` flag used with upstream configured

## Golden Rules

1. **Always backup** - `.codecrafter.bak.*` files retained until explicit cleanup
2. **Never mix formatting and refactoring in same pass** - separate phases prevent unreviewable diffs
3. **Respect existing configs** - read project-level config before applying defaults
4. **Never remove code marked `# noqa`**, `// @ts-ignore`, or `// codecrafter:skip`
5. **Always preserve imports order** recognized by isort/profile
6. **Exit 0 only if all verifications pass** when `--check` or `--strict` mode
7. **Require `--force` for destructive refactorings** (inline-variable, rename-symbol)
8. **Log every action to `.codecrafter.log`** with full command and file hashes
9. **Reject modifications to generated files** (protobuf, thrift, `*-generated.go`, `dist/`, `build/`)
10. **Suggest manual review** for complexity reductions > 20% or duplicate removals

## Examples

### Example 1: Format Python with Google style
```
$ code-crafter format src/ --language python --style google --dry-run
[DRY RUN] google style, line-length=88
✓ src/utils.py: reformatted (was 142 lines → 138 lines)
✗ src/models.py: no changes needed
Verification: python -m py_compile on reformatted files: PASSED
Run without --dry-run to apply
```

### Example 2: Extract function from repetitive block
```
$ code-crafter extract-function src/auth.py --region 45:67 --name validate_token_payload
Extract 23 lines into validate_token_payload()
Created new function at line 45
Updated 3 call sites
Backup: src/auth.py.codecrafter.bak.20260305_142310
```

### Example 3: Fix imports and lint TypeScript
```
$ code-crafter fix-imports src/components/ --profile google
$ code-crafter lint src/components/ --fix --ignore "N801,N802"
Fixable lint errors: 12/15 applied
Remaining: 3 (non-fixable naming violations)
```

### Example 4: Find and report complex functions
```
$ code-crafter complexity lib/ --threshold 10 --format json
[
  {
    "file": "lib/processor.py",
    "function": "process_transaction",
    "line": 234,
    "complexity": 18,
    "suggested_threshold": 10
  }
]
```

### Example 5: Batch operations via YAML
```
$ cat batch.yaml
- format:
    path: src/
    language: python
    style: pep8
- lint:
    path: src/
    fix: true
- complexity:
    path: src/
    threshold: 15
    format: json
$ code-crafter batch batch.yaml --report batch-report.json
Applied 3 operations
Report: batch-report.json
```

## Rollback Commands

### Per-file immediate revert
```
$ code-crafter rollback <file> [--to-backup <timestamp>]
# Restores latest .codecrafter.bak.* unless specific timestamp given
```

### Git-based revert
```
$ git checkout -- <file>  # if not yet committed
$ git revert <commit_sha>  # if already committed
```

### Batch rollback all Code Crafter changes
```
$ code-crafter rollback-all --since "2 hours ago"
# Finds all files with .codecrafter.bak.* and reverts
# WARNING: unsaved work may be lost
```

### Cleanup backups
```
$ code-crafter cleanup-backups --older-than 7d
$ code-crafter cleanup-backups --all
```

## Troubleshooting

**Issue**: "Language not supported"
- Fix: Install required formatter (e.g., `npm install -g prettier`, `apt install clang-format`)
- Check: `code-crafter --list-languages`

**Issue**: Conflict between formatters (e.g., Black then isort reorders)
- Fix: Set `--profile black` for isort or use `pyproject.toml` to configure both

**Issue**: "Refactoring changed semantics"
- Restore from backup: `code-crafter rollback <file>`
- Re-run with `--dry-run` to inspect diff before applying
- Report bug with minimal repro case

**Issue**: "Verification failed: tests broken"
- Abort: `code-crafter abort` (cancels current batch)
- Investigate: `code-crafter diff <file>` shows pre/post changes
- Use `--skip-tests` only for cosmetic format-only runs

**Issue**: "Skipping generated file"
- Add `--allow-generated` to override (use with caution)
- Or mark file with comment: `// codecrafter: allow-generated`

## Environment Variables

- `CODECRAFTER_BACKUP_DIR`: Custom backup location (default: same directory)
- `CODECRAFTER_MAX_BACKUPS`: Number of backups to retain (default: 5, 0=unlimited)
- `CODECRAFTER_VERBOSE`: Set `1` for detailed logging
- `CODECRAFTER_GIT_INTEGRATION`: Set `0` to disable auto-staging/committing
- `CODECRAFTER_STYLE_OVERRIDE`: Force style for all languages (e.g., `google`)

## Exit Codes

- 0: Success (all verifications passed)
- 1: Changes needed (with `--check`)
- 2: Lint/format errors remain after `--fix`
- 3: Syntax/compilation error
- 4: Test suite failure
- 5: Backup/restore failure
- 6: Unsupported language or missing dependency
- 7: User abort (e.g., keyboard interrupt)

## Logging

All operations append to `.codecrafter.log`:
```
[2026-03-05 14:23:01] START: format src/ --style google
[2026-03-05 14:23:02] BACKUP: src/utils.py → src/utils.py.codecrafter.bak.20260305_142302
[2026-03-05 14:23:03] FORMAT: src/utils.py (black)
[2026-03-05 14:23:04] VERIFY: py_compile PASSED
[2026-03-05 14:23:04] END: format SUCCESS
```

## File Patterns (Reserved)

These patterns are **always skipped** unless `--force`:
- `dist/`, `build/`, `target/`, `node_modules/`, `__pycache__/`
- `*.min.js`, `*.pyc`, `*.o`, `*.so`
- `*-generated.go`, `*.pb.go`, `pkg/gen/`
- `vendor/`, `third_party/`, `external/`

## Performance Notes

- Format: ~50 files/sec (Python), ~200 files/sec (JS/TS)
- Refactor: ~10 operations/sec (AST-bound)
- Use `--parallel N` to override concurrency (default: CPU count)
- For > 1000 files, run per-directory to avoid memory pressure

## Safety Checklist

Before running destructive refactoring:
1. `git status` clean or changes committed
2. Tests passing on current code
3. `--dry-run` reviewed diff manually
4. Backup confirmed in log
5. CI/CD pipeline aware of potential changes

```