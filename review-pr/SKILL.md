---
name: review-pr
description: Use this skill when the user asks to review the current PR or branch — e.g. "review this PR", "review my branch", "do a code review", "check this PR for issues". Acts as a senior engineer reviewing the current branch/PR with focus on security, bugs, and maintainability, then posts the review to the GitHub PR.
version: 0.1.0
---

# Review PR

Act as a senior software engineer reviewing the current branch / PR. Focus on **security vulnerabilities, bugs, and maintainability** — not style nits or bikeshedding. Surface issues that could ship to production and cause real problems.

## Scope

Review **only the current branch / PR**, not all open PRs on the repo.

Resolve the current PR by:
1. Run `gh pr view --json number,title,body,headRefName,baseRefName,url,author` from the working directory. If this fails because there's no PR for the current branch, tell the user and stop.
2. If multiple PRs match, ask the user which one.

## Procedure

### 1. Gather context

Run in parallel:
- `gh pr view <num> --json number,title,body,headRefName,baseRefName,url,author,commits` — PR metadata + commit list
- `gh pr diff <num>` — the diff to review
- `gh pr view <num> --comments` — existing review comments

### 2. Understand the codebase

Before writing comments, explore the surrounding code so suggestions align with existing patterns:
- Read the files touched by the diff in full (not just the hunks) so review context isn't truncated.
- Grep for existing conventions when the diff introduces something new (error handling, validation, auth checks, logging, test patterns).
- Note framework / language idioms in use — don't suggest patterns foreign to the codebase.

### 3. Review with a senior engineer's lens

Prioritize, in order:

**Security** (highest priority)
- Injection: SQL, command, template, XSS, SSRF, path traversal, prototype pollution
- AuthN / AuthZ: missing checks, broken access control, IDOR, privilege escalation
- Secrets / credentials: hardcoded keys, tokens logged, secrets in error messages
- Input validation & sanitization at trust boundaries
- Unsafe deserialization, regex DoS, race conditions in security-sensitive code
- Dependency risks introduced by the change

**Bugs / correctness**
- Logic errors, off-by-one, incorrect conditionals, wrong operator
- Null / undefined / unset handling
- Error handling: swallowed errors, wrong error type, missing rollback
- Concurrency issues: races, deadlocks, missing locks, unawaited promises
- Resource leaks: unclosed handles, connections, listeners
- Edge cases the diff missed (empty input, large input, unicode, timezones)

**Maintainability**
- Dead code, duplicated logic that should be reused
- Functions / files that have grown too large or mixed concerns
- Confusing names, unclear control flow, magic numbers without context
- Missing or misleading types in typed codebases
- Tests: missing coverage for new branches, brittle assertions, tests that don't actually test the change

Skip nits (formatting, single-letter variable preferences, subjective style) unless they actively harm readability.

### 4. Write the review

Post a single PR review (not scattered individual comments) with:

**Summary section** — a few bullets covering:
- What the PR does (one or two sentences)
- The codeflow / how the changes fit together, with links to specific lines (e.g. `path/to/file.ts:42`)
- Overall assessment

**Findings** — one item per issue. Each finding must:
- Use a [Conventional Comments](https://conventionalcomments.org/) label: `praise:`, `nitpick:`, `suggestion:`, `issue:`, `question:`, `thought:`, `chore:`, `note:`. Add decorations where useful (`(security)`, `(non-blocking)`, `(if-minor)`).
- Reference the exact file and line (`path/to/file.ts:L42-L48`).
- Explain *why* it matters, not just *what* to change.
- End with a confidence rating: `Confidence: high | medium | low` — indicating how likely the comment is genuinely relevant given codebase context you may not have.

Example finding:

```
**issue (security):** `path/to/handler.ts:L88` — the `userId` query param is passed straight to `db.query` without parameterization. Even though the caller is internal today, this becomes an SQL injection vector if the endpoint is ever exposed. Use the existing `db.prepared(...)` helper used in `other/file.ts:L210`.

Confidence: high
```

### 5. Post the review

Use `gh pr review <num> --comment --body-file -` with the body piped via heredoc. Use `--comment` (not `--approve` or `--request-changes`) — Claude is assisting, not gating merge.

Also print the full review body to the user in the chat so they can see what was posted.

## Tone

- Direct, specific, senior-engineer voice. No flattery, no filler.
- Avoid hedging on real issues; do hedge (via `Confidence: low/medium`) when codebase context is incomplete.

## What this skill does NOT do

- Does not save the review to disk — it lives on the PR.
- Does not review other open PRs — current branch only.
- Does not approve or request changes — it leaves a `--comment` review only.
