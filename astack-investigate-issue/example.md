# Table drag handles lag on scroll · `bug`

## Headline
- **Likelihood:** **medium** — P1 fits symptom (“stale until click”) and code path; not reproduced in session (`measure.ts:164-172`, `useTableGeometries.ts:69-71`)
- **Why:** Scroll runs `measure`, but unchanged rects → `reconcileGeometries` keeps prior state → handles may not repaint; cell click → `transaction` → remeasure aligns UI
- **Fix (minimal):** Coalesce remeasure on existing `containerRef` scroll handler (e.g. rAF); only if needed, bypass dedupe on scroll-sourced measure
- **Lineage:** **pre-existing (default)** — reconcile-on-scroll behavior in `measure.ts` / `useTableGeometries.ts` unchanged in `git diff main...HEAD` for those paths (verify on your default branch name)

## Hypotheses
| | Hypothesis | Conf | Notes |
|--|------------|------|-------|
| P1 | Dedupe skips commit on scroll | med | coord math stable; listener fires |
| P2 | Scroll on ancestor, not container | med | no listener on outer scroll parent |
| — | *Ruled out* no scroll handler | — | capture listener on `containerRef` |

## Why tests missed it
| Test | Asserts | Gap |
|------|---------|-----|
| `TableDragHandles.browser.test.tsx` | resize → remeasure | no scroll |
| `TableBubbleMenu.browser.test.tsx` | menu on scroll | not applied to handles |

## Minimal fix (analysis only)
| Scope | Change |
|-------|--------|
| Smallest (if P1) | Coalesce remeasure on existing `containerRef` scroll handler (e.g. rAF)—same hook, no new listeners |
| Next (if P1 insufficient) | Bypass `reconcileGeometries` dedupe for scroll-sourced measure—not ancestor scroll parents until P2 confirmed |
| Regression | More scroll-time renders |
| Falsifier | Drift while reconcile output unchanged |

## Tests to add
| File | Case | Assert |
|------|------|--------|
| `TableDragHandles.browser.test.tsx` | vertical scroll | set `containerRef` `scrollTop`; row handle within 1px of cell |
| `TableDragHandles.browser.test.tsx` | horizontal scroll | scroll `[data-table-scrollport]`; column handles track `frame.left` |
