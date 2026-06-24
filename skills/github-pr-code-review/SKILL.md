---
name: github-pr-code-review
description: Review a GitHub pull request, reconcile prior inline review threads, resolve fixed findings, and post new inline comments without editing code or managing the PR.
---

# GitHub PR code review

## Role

Review the requested GitHub pull request and manage only this workflow's inline review comments. Use the general `code-reviewer` skill priorities, checklist, and output format when evaluating code.

The caller will provide:

- Pull request number.
- Repository in `owner/repo` form.
- Head SHA.

This run reconciles prior review threads on every push, including bot commits. Do not stop early because the bot commented before.

## Stable finding IDs

For every finding you may post, compute a stable marker id before posting:

- `marker_id = sha256("<relative-file-path>:<one-line issue summary>")` truncated to 12 hex chars, lowercase.
- Embed in every inline comment body on its own line: `<!-- ai-code-review: <marker_id> -->`
- The one-line issue summary must be stable across runs, using the same wording for the same underlying issue.

## Step 1 - Load prior review threads

Run this GraphQL query via `gh api graphql`, paginating with `after` if `hasNextPage` is true:

```graphql
query($owner: String!, $repo: String!, $pr: Int!, $after: String) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100, after: $after) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          isResolved
          path
          line
          comments(first: 20) {
            nodes {
              body
              author { login }
            }
          }
        }
      }
    }
  }
}
```

Variables: `owner` and `repo` from the provided repository, split on `/`; `pr` is the provided pull request number.

Build a list of **prior findings** from threads whose first comment body contains an `<!-- ai-code-review:` marker. For each thread record:

- `thread_id`, the GraphQL node `id`.
- `isResolved`.
- `marker_id`, parsed from `<!-- ai-code-review: ... -->` in the body, if present.
- `path`, `line`, and a short summary from the comment body.

Also run `gh pr view <pr-number>` for title and description.

## Step 2 - Determine review scope

Decide whether this is an **initial** or **incremental** review based on the prior review threads from Step 1:

- **Initial review**: there are zero prior review threads on this PR, meaning no thread, resolved or unresolved, carries an `<!-- ai-code-review:` marker. New findings may be posted for anywhere in the PR diff.
- **Incremental review**: one or more prior review threads exist. New findings may only be posted for issues that the latest commits explicitly introduced.

If this is an incremental review, determine the last reviewed commit and the changes added since then:

1. Fetch prior review comments and their commit anchors:

```bash
gh api repos/<owner>/<repo>/pulls/<pr-number>/comments --paginate
```

1. Among comments whose body contains `<!-- ai-code-review:`, pick the one with the most recent `created_at`. Use its `original_commit_id` as `last_reviewed_sha`.
1. If no such comment exists or `last_reviewed_sha` cannot be resolved, fall back to treating this run as an initial review.
1. Otherwise compute the incremental change set, meaning the files and line ranges introduced by commits pushed since the last review:

```bash
git diff <last_reviewed_sha>..<head-sha> --name-only
git diff <last_reviewed_sha>..<head-sha>
```

"Latest commits" means all commits pushed since `last_reviewed_sha`, not only the single newest commit. Record the incremental change set; it gates new findings in Step 5.

## Step 3 - Review current diff

Run `gh pr diff <pr-number> --name-only`, then `gh pr diff <pr-number>`.

Apply the `code-reviewer` skill priorities, checklist, and output format. Produce a list of **current findings** with severity, file, line, summary, explanation, and `marker_id`. Always review the full PR diff here so prior threads can be reconciled correctly, regardless of review scope.

## Step 4 - Reconcile prior threads vs current findings

Match by `marker_id` first. If a prior thread has no marker, fall back to same `path` plus overlapping line range plus similar summary.

For each unresolved prior thread:

- If no current finding matches, meaning the issue was fixed, removed, or no longer applies, resolve it with:

```graphql
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) {
    thread { isResolved }
  }
}
```

via `gh api graphql -f query='...' -f threadId='<thread_id>'`.

- If a current finding still matches: leave the thread open and do not post a duplicate comment.

Reconciliation always runs against the full-diff current findings, so issues fixed by the latest commits are resolved and still-valid prior issues stay open, even when this is an incremental review.

## Step 5 - Post new inline comments

For each current finding whose `marker_id` is not already present on an unresolved prior thread:

- Incremental reviews only: skip the finding unless its file and line fall within the incremental change set computed in Step 2. Do not open a new thread for a finding outside the latest changes, even if it is a genuine issue elsewhere in the PR.
- Post exactly one inline review comment anchored to the relevant line via `gh api`:

```bash
gh api repos/<owner>/<repo>/pulls/<pr-number>/comments \
  -f body="$BODY" \
  -f commit_id='<head-sha>' \
  -f path='<relative-file-path>' \
  -F line=<line-number> \
  -f side='RIGHT'
```

Use the provided head SHA as `commit_id`, and anchor `line` to a line present in the changed diff for all severities: Critical, High, Medium, and Low. For a multi-line range, also pass `-F start_line=<n>` and `-f start_side='RIGHT'`.

- Include in the body: the `<!-- ai-code-review: <marker_id> -->` line, severity label, what is wrong, why it matters, and the smallest suggested fix.
- For small self-contained fixes of five lines or fewer in a single location, include a committable GitHub suggestion block in the body.
- Never post a monolithic rollup PR comment listing multiple findings.

If a finding cannot be anchored to a line in the diff, skip posting it and do not fall back to a top-level PR comment for that finding.

## Step 6 - No new comments to post

If there are no new findings to post:

- Resolve any remaining unresolved prior review threads that no longer apply.
- Do not post a recurring "no issues found" comment on every push.
- Only if there are zero prior review threads on this PR, you may post one top-level comment: `**Code review**: No issues found. Checked behavior, tests, organization, and code quality.`

## Hard rules

- Use `gh` CLI, including `gh api graphql`, for all GitHub access. Do not web-fetch.
- Never edit files, push commits, create branches, or open/update other PRs.
- Never approve, request changes, or merge the PR.
- Do not run the full test suite, linter, or type-checker. Run only read-only verification commands when useful; skip expensive or environment-dependent checks and note that in output.
- One inline thread per unique finding; never duplicate an open thread for the same `marker_id`.
- On subsequent incremental reviews, never open a new inline thread for an issue outside the changes introduced since `last_reviewed_sha`, even if it is a real issue elsewhere in the PR.
