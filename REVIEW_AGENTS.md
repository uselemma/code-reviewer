# Repository Review Guide

This file gives the automated reviewer repository-specific context. Keep it short, concrete, and biased toward rules that change whether a PR should receive a finding.

## Commands

Document the fastest focused checks first. Prefer package-local commands over full-repo commands when a PR touches one area.

```bash
# Examples. Replace these with this repository's real commands.
npm run lint --workspace <package>
npm run typecheck --workspace <package>
npm test --workspace <package> -- <test-file>
```

If a command requires secrets, services, large fixtures, or long-running infrastructure, say so here. The reviewer should not run it automatically.

## Review Rules

- Report correctness, security, reliability, build, type, test, and maintainability problems.
- Do not report formatter-only style issues unless they reveal a missing or broken formatter/linter rule.
- Prefer findings that point to a concrete failure mode and a small fix.
- Do not approve, request changes, merge, push commits, or edit files.
- Do not post top-level summaries when inline comments can be anchored to the changed line.

## Local Conventions

Replace this section with conventions that matter in this repository:

- Public API and import-path compatibility rules.
- Naming, file placement, and ownership boundaries.
- Required tests for bug fixes, new features, migrations, and refactors.
- Generated-code, vendored-code, lockfile, snapshot, and fixture policies.
- Framework-specific requirements such as directives, route conventions, or build constraints.

## Generated And External Files

By default, do not review generated, vendored, minified, or third-party files for style or structure. Only comment on them when the PR changes how they are produced, checked in, loaded, or secured.

## Expensive Checks

If a check is too slow or flaky for automated review, list it here with the safer alternative. The reviewer should mention skipped checks only when they are relevant to the risk in the PR.
