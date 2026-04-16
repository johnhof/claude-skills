# implement-review

Stage 4 of the split implement workflow. Runs ONE review iteration against the solution: syncs with main, reviews the diff, and if changes are requested, applies fixes. Re-run this skill for each iteration until approved (max 5 total).

Prior stage: `implement-select` | Next stage: either re-run `implement-review` or proceed to `implement-finalize`

---

## Resume / state check

Read `~/.claude/runs/implement/.current` to get the active slug. Then read STATE.json.

- If `step` is `select_done`: this is iteration 1 — proceed
- If `step` matches `review_N_done` and last status was `CHANGES_REQUESTED`: this is iteration N+1 — proceed
- If `step` is `approved`: already approved — print status and exit (run `implement-finalize` next)
- If `review_iteration` is already 5: print `REVIEW_STATUS: MAX_ITERATIONS_REACHED`, list unresolved findings, and exit
- Otherwise: print an error — wrong stage — and exit

Determine current iteration number: `N = review_iteration + 1`.

Print:
```
[review] ┌ Iteration N of 5 — slug: <slug>
[review]   solution → <absolute-path-to-solution-dir>
```

---

## What to do

### 1. Sync with main

In the solution worktree (`~/.claude/runs/implement/<slug>/solution/<project>`):

Print `[sync] ├ START  merging origin/main`

```bash
git -C <solution-path> fetch origin && git -C <solution-path> merge origin/main
```

Resolve conflicts:
- Files touched by the implementation: prefer implementation's version; re-apply unrelated semantic changes from main
- Files not touched: prefer main
- Ambiguous: prefer main, print `[sync] ├ WARNING ambiguous conflict in <file> — preferred main`

Print `[sync] ├ DONE   no conflicts` or `[sync] ├ DONE   N conflicts resolved`

### 2. Review

Print `[reviewer] ├ START  scanning diff + codebase context`

Launch a reviewer agent that:
- Reviews **only the changed files and lines in this diff** — not pre-existing code
- Uses `~/.claude/CODE_REVIEW.md` as rubric (if it exists; otherwise apply standard code review criteria)
- Every finding must cite a specific file and line from the diff — no findings about pre-existing code
- **Runs CI** (build, lint, tests, coverage) in the solution worktree; any CI failure is an automatic Blocker
- Coverage on new code below 80% is an automatic Major
- Writes structured report to `~/.claude/runs/implement/<slug>/reviews/iteration-N/review.md`
  Format: CI results header → Blockers → Majors → Minors → Nits
- Declares `APPROVED` or `CHANGES REQUESTED`

Print:
```
[reviewer] ├ DONE   APPROVED
[reviewer] ├        report → <absolute-path>
```
or:
```
[reviewer] ├ DONE   CHANGES REQUESTED — N Blockers, N Majors, N Minors
[reviewer] ├        report → <absolute-path>
```

### 3a. If APPROVED

Update STATE.json: `"step": "approved"`, `"review_iteration": N`

Print `REVIEW_STATUS: APPROVED` (machine-readable)

Print `[review] └ APPROVED after N iteration(s) — run implement-finalize to create the PR`

Write GitHub step summary (see below).

Exit.

### 3b. If CHANGES REQUESTED

Print `[fixer] ├ START  addressing N Blockers, N Majors`

Launch a fixer agent that:
- Implements all Blocker and Major findings
- Re-runs CI after fixes; resolves any CI failures before completing
- Keeps coverage ≥ 80% on new code
- Writes fix summary to `~/.claude/runs/implement/<slug>/reviews/iteration-N/changes.md`

Print:
```
[fixer] ├ ci     build: PASS | lint: PASS | tests: PASS (coverage: XX%)
[fixer] ├ DONE   changes → <absolute-path-to-changes.md>
```

Update STATE.json: `"step": "review_N_done"` (where N is current iteration), `"review_iteration": N`

Print `REVIEW_STATUS: CHANGES_REQUESTED` (machine-readable)

Print `[review] └ DONE   iteration N complete — re-run implement-review for iteration N+1`

### Write GitHub step summary

If `$GITHUB_STEP_SUMMARY` is set, append:

```markdown
## Review — Iteration N

**Status:** APPROVED ✓ | CHANGES REQUESTED ✗

**CI:** build: PASS | lint: PASS | tests: PASS (coverage: XX%)

### Findings
- **[Blocker]** `file.ts:42` — <description>
- **[Major]** `file.ts:87` — <description>

### Fixes Applied _(if CHANGES REQUESTED)_
- <brief description of each fix>

[Full report](<absolute-path-to-review.md>)
```

---

## Notes
- This skill runs exactly ONE iteration. The autofix workflow loops by re-running it until `REVIEW_STATUS: APPROVED`.
- Minor and Nit findings are logged in review.md but are NOT fixed by the fixer agent — only Blockers and Majors
- Each iteration must not re-raise findings already resolved in prior iterations
- The fixer must not introduce new issues while resolving existing ones
