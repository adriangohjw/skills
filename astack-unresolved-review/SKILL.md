---
name: astack-unresolved-review
description: Use when a pull request has unresolved review threads or open feedback and you must judge whether comments are technically valid before changing code. Use when asked to triage review comments, assess reviewer feedback, or summarize what reviewers flagged without implementing fixes or posting replies.
---
# Assess Unresolved Review
## Overview
Read-only triage of **unresolved** PR review feedback. Technical evaluation only—verify against the codebase, question scope, check YAGNI—then produce an analysis document.
**Core principle:** Verify before treating feedback as work. External comments are suggestions to evaluate, not orders. Technical correctness over social comfort. This skill stops at the report: no implement, no GitHub respond.
## When to Use
- Unresolved inline review threads on the current PR
- Partner asks to “validate review comments”, “triage feedback”, or “what’s actually wrong?” without fixing yet
- Before a large pass implementing review feedback—need grouped themes and minimal fix scope
**When NOT to use**
- Implementing fixes → `bugfix` / normal development
- Posting reviews or replies → `review-pr` / manual `gh`
- Full PR grading without existing reviewer comments → `pr-review`
## Hard Rules (Non-Negotiable)
| Allowed | Forbidden |
|---------|-----------|
| `gh`, `git`, read files, grep, run tests **only to verify claims** | Edit source, commit, push |
| Print analysis to chat | `gh pr review`, `gh pr comment`, thread replies |
| Mark comment **valid / partially valid / invalid** with evidence | “Fixing now”, “Posted reply”, performative agreement |
If a comment is wrong, say so in the **Validity** line—do not implement “just to be safe.”
**In the report, never:** performative agreement (“great point”, “reviewer is absolutely right”), gratitude, or hedging that implies you will implement without a separate user request.
## Evaluating feedback (per thread)
Use this loop for every unresolved thread. Steps 1–4 only; this skill does not implement or post replies.
```
1. READ:    Full thread (all comments in the thread), no reacting
2. UNDERSTAND: Restate the claim in one technical sentence; if unclear, mark
               cannot verify and list what clarification would be needed (do not guess)
3. VERIFY:  Code, types, tests, runtime if needed, AGENTS.md / conventions
4. EVALUATE: Technically sound for THIS codebase? Verdict + evidence
```
**Before marking valid**, check:
1. Technically correct for this stack and repo patterns?
2. Would the suggested change break existing behavior?
3. Is there a documented reason for the current implementation?
4. Platform/version constraints?
5. Does the reviewer likely lack context you found in full-file read?
**External reviewers:** skeptical, but check carefully. Wrong suggestions get `invalid` with reasoning—not softened to “partially valid” to avoid conflict.
**YAGNI** when reviewers ask to “implement properly”:
```
grep for callers / usage
IF unused or dead path → verdict often invalid or “fix = remove unused code”, not expand scope
IF used → minimal fix scoped to real usage only
```
**Push back in the report** (verdict `invalid` or narrowed `partially valid`) when:
- Suggestion breaks existing functionality
- Reviewer lacks context you verified in code
- Violates YAGNI (unused feature, hypothetical edge)
- Technically wrong for this stack
- Legacy/compatibility requires current shape
- Conflicts with your partner’s stated architecture (note and defer)
**Unclear feedback:** if any part of a thread is ambiguous, do not fold it into a confident minimal fix. Separate “needs clarification” from “valid once clarified.”
**Cannot verify:** verdict `cannot verify without <X>`; do not invent a fix. Say what investigation would resolve it.
## Inputs
- Current branch’s PR (default). If no PR: stop and report.
- Optional: `gh pr view <n>` when not on the PR branch.
## Procedure
### 1. Resolve the PR
```bash
gh pr view --json number,title,url,headRefName,baseRefName
```
If multiple PRs match, ask which number. Do not proceed without a PR.
### 2. Fetch unresolved review threads only
Use GraphQL (`gh pr view` does not expose thread resolution). Paginate until `reviewThreads.pageInfo.hasNextPage` is false.
```bash
OWNER=$(gh repo view --json owner -q .owner.login)
NAME=$(gh repo view --json name -q .name)
NUMBER=<pr-number>
gh api graphql -f query='
query($owner: String!, $name: String!, $number: Int!, $after: String) {
  repository(owner: $owner, name: $name) {
    pullRequest(number: $number) {
      reviewThreads(first: 100, after: $after) {
        pageInfo { hasNextPage endCursor }
        nodes {
          isResolved
          path
          line
          diffSide
          comments(first: 50) {
            nodes {
              body
              path
              originalLine
              line
              author { login }
              createdAt
            }
          }
        }
      }
    }
  }
}' -f owner="$OWNER" -f name="$NAME" -F number="$NUMBER" -f after="$CURSOR"
```
- **Include** threads where `isResolved` is `false`.
- **Exclude** resolved threads and general PR conversation unless the user explicitly asked for those too.
- Preserve **file**, **line**, **author**, and **verbatim comment** (or tight quote) for traceability.
If there are zero unresolved threads, say so and stop—do not invent work.
### 3. Load change context
In parallel where possible:
- `gh pr diff <number>`
- Full files touched by unresolved comments (not only diff hunks)
- Existing tests adjacent to flagged code (`__tests__`, `*.test.ts`, E2E paths per repo conventions)
- Relevant `AGENTS.md` for touched directories
### 4. Verify each thread
Apply **Evaluating feedback** above. Record per thread:
- Restated claim
- Verdict: `valid` | `partially valid` | `invalid` | `cannot verify without <X>`
- Evidence: `path:line`, test run, or grep result
Do not batch-implement. Do not reply on GitHub.
### 5. Group into themes
Cluster threads by **root cause**, not by file (one theme may span files). Examples: missing auth check, incorrect error handling, test gap on edge case, API contract mismatch, race/timing, misleading types.
Every thread must appear in exactly one theme section (or an explicit **Uncategorized** section with reason).
### 6. Write the report
Use the output format below. Order themes by severity: security/correctness first, then maintainability, then nits (still list nits if unresolved—mark validity honestly).
## Output Format
```markdown
# Unresolved review assessment — PR #<n> <title>
**PR:** <url>
**Unresolved threads:** <count> (**valid:** n, **partially valid:** n, **invalid:** n, **unverified:** n)
**Scope:** Read-only; no code or GitHub actions taken.
