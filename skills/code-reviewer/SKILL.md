---
name: code-reviewer
description: Review code changes across a repository and report findings without editing, committing, or managing PRs. Use when evaluating a branch, pull request, or diff for behavior, tests, organization, code quality, readability, duplication, maintainability, and security.
---

# Code reviewer

## Role

Act as a reviewer, not an implementer. Inspect the relevant diff and surrounding code, then report actionable findings. Do not edit files, create commits, push branches, or open/update PRs while using this skill unless the user explicitly asks you to switch from review to implementation.

## Review priorities

Focus on issues that could affect correctness, behavior, reliability, security, performance, maintainability, or future reviewability.

Prioritize:

1. Behavioral regressions or mismatches with the old code.
2. Type, lint, formatting, or build failures introduced by the change.
3. Missing or weakened tests for changed behavior.
4. Public API or import path breakage.
5. State management, async, error handling, race condition, and lifecycle bugs.
6. Security/privacy issues, especially secrets, auth, permissions, and user data.
7. Code smells, readability issues, unnecessary complexity, or unclear ownership boundaries.
8. Duplicated logic, repeated constants, parallel implementations, or copy-paste branches that should share a helper or abstraction.
9. File organization problems that make changes harder to review or maintain.
10. Inconsistent naming, helper placement, or local convention drift.

Avoid spending review budget on style nits that an existing formatter or linter already handles unless they indicate a real convention or tooling gap.

## How to review

1. Identify the base and changed branch or diff.
2. Read the changed files and enough neighboring code to understand the behavior.
3. Compare extracted or moved code against the original behavior when the change is a refactor.
4. Check package-local scripts and config to understand expected lint, format, type-check, and test commands.
5. Run read-only verification commands when useful. If commands are expensive or environment-dependent, say what was not run and why.
6. Report findings with file/line references and concrete failure modes.

## Output format

Lead with findings, ordered by severity.

For each finding include:

- Severity: `Critical`, `High`, `Medium`, or `Low`.
- File and line reference.
- What is wrong.
- Why it matters.
- The smallest suggested fix or direction.

Then include, only if useful:

- Open questions or assumptions.
- Tests/checks run.
- Residual risk or areas not reviewed.

If there are no findings, say that clearly and mention any checks you ran or remaining test gaps.

## Review checklist

### Behavior and API

- Does the new code preserve existing user-visible behavior?
- Did refactors move logic without changing lifecycle, state reset, animation, async, or error-handling semantics?
- Are public exports and import paths still stable?
- Are feature flags, env vars, auth checks, and permission boundaries preserved?

### Tests and verification

- Are changed behaviors covered by tests?
- Are snapshots and fixtures updated intentionally?
- Do package-local type-check, lint, format, and relevant test commands pass?
- If new warnings are introduced, are they intentional and visible?

### Organization and naming

- Are files split along meaningful ownership and responsibility boundaries?
- Are helpers/constants placed near their main callers unless shared use justifies a wider location?
- Are generated, vendored, and framework-owned files left alone?
- Are suppressions avoided or justified by local policy?

### Code quality and maintainability

- Is the code easy to read at the call site, with clear names and straightforward control flow?
- Are conditionals, async flows, and state transitions understandable without tracing many hidden side effects?
- Is there avoidable duplication between branches, components, tests, queries, or prompts?
- Would a small helper, shared constant, or existing local abstraction reduce real duplication without obscuring behavior?
- Are abstractions proportional to the problem, or did the change introduce premature generality, indirection, or overly generic naming?
- Are functions, classes, and modules cohesive, or do they mix unrelated responsibilities?
- Are error messages and logging useful for debugging without leaking secrets?

### Repository-specific reminders

Add your repository's local review rules here, such as:

- Required package-local lint, type-check, format, and test commands.
- Framework directives or file conventions that reviewers should preserve.
- Generated code, migration, or snapshot policies.
- Known expensive or environment-dependent checks that should usually be skipped in automated review.
