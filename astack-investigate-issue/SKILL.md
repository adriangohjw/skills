---
name: astack-investigate-issue
description: Use only when the user explicitly invokes or attaches astack-investigate-issue (e.g. @astack-investigate-issue). Not for ambient bug triage without that invocation.
disable-model-invocation: true
---

# Investigate reported issue

**Invocation:** Run this skill only when the user explicitly attached it or named `astack-investigate-issue`. Do not auto-apply for generic “investigate this bug” messages.

Read-only: investigate in code/tests/repro. **No fix, commits, or outward posts** unless the user separately asks.

**Input:** pasted free text only; pointer without body → ask for paste; do not fetch.

## When not to use

- User asked to implement the fix
- Mid-fix debugging with instrumentation (past investigation)
- Code-review threads without an end-user symptom report

| Do | Don't |
|----|--------|
| Full investigation before writing | Prose paragraphs or repeating the same fact |
| **Output:** same sections, order, table columns as `example.md` | Extra sections or long essays |
| Replace all example paths/names with this repo’s facts | Copy RTE sample literals into unrelated reports |
| `path:line` on technical claims | Vague references |
| **~30–45 lines** total | Long-form unless user asks |

## Investigation (not pasted)

Normalize → classify → repro (≤3 UI tries) → **lineage** (below) → trace in notes with `path:line` → why tests missed it (`bug`/`regression`) → P1–P3 + ruled out → minimal fix tiers → tests to add (with file paths).

**Lineage (default branch vs current branch):** Establish whether the failure is **pre-existing on the default branch** or **introduced on the current branch** — do not guess.

- Repro or read suspect paths on default when feasible (`git show <default>:<path>`, temporary checkout, or same steps on a default-branch workspace).
- Compare suspect files: `git diff <default>...HEAD -- <paths>`; note commits that touch the divergence (`git log <default>..HEAD --oneline -- <paths>`).
- If repro only on current branch → **introduced (current branch)** with diff/commit cite. If same on default → **pre-existing (default)**. If not checked → **unknown** and say what was missing.
- `regression` classification usually implies branch introduction; still verify — latent bugs can exist on default without being new code.

## Report

**Required:** Open and follow `example.md` — same `#` title pattern (`# <title> · `<classification>`` only — no repro status in the title), same `##` section headings in order, same table column headers. Only the example’s *content* is domain-specific; do not invent alternate layouts.

Sections (exact headings): Headline → Hypotheses → Why tests missed it → Minimal fix (analysis only) → Tests to add.

- **Headline** includes **Likelihood**, **Why**, **Fix (minimal)**, and **Lineage** (see investigation). Put repro status in **Likelihood** when relevant, not in the title. Put key `path:line` cites in **Why** and **Likelihood**.
- Headline **Fix (minimal)** must match minimal-fix → **Smallest (if P1)**.
- One fact per section; avoid repeating the same pointer in Headline and Hypotheses.
- Classification **`insufficient-info`:** omit minimal-fix and tests-to-add tables; state gaps in **Likelihood**.

## Red flags — stop

- Editing source “while investigating”
- Fetching unpasted ticket content
- Report longer than ~45 lines without user request
- Copying the example’s file names into a different codebase report

| Excuse | Reality |
|--------|---------|
| “I’ll fix while I’m in there” | Investigation stops at the report unless user asked to fix |
| “Example used these paths” | Example shows **shape** only; paths must come from this investigation |
| “One more section won’t hurt” | Extra sections caused bloat; follow the example |
