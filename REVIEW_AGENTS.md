# Review Instructions

This file gives the automated reviewer repository-local context. Keep it short and focused on rules that materially affect review quality.

## Useful Commands

Update these examples for your repository:

```bash
npm test
npm run lint
npm run typecheck
```

## Review Guidance

- Prefer package-local commands and configuration over repository-wide checks when reviewing a focused change.
- Treat generated, vendored, and framework-owned files according to your repository policy.
- Avoid posting findings for formatter-only style issues if a formatter or linter already enforces them.
- Do not run expensive or environment-dependent commands unless the repository documents them as safe for CI review.

## Repository Conventions

Add conventions that are specific to your codebase, such as:

- Naming and file organization rules.
- Required tests for common change types.
- Migration, generated code, and snapshot update policy.
- Framework-specific directives or build constraints.
