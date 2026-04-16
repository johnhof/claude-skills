# implement-finalize

Stage 5 (final) of the split implement workflow. Renames the solution branch, pushes it, opens a draft PR, and prints the implementation summary.

Prior stage: `implement-review` (must be APPROVED) | No next stage

---

## Resume / state check

Read `~/.claude/runs/implement/.current` to get the active slug. Then read STATE.json.

- If `step` is `approved`: proceed
- If `step` is `finalized`: already done — print the PR URL and exit
- Otherwise: print an error — wrong stage (review must be APPROVED first) — and exit

Print:
```
[finalize] ┌ Finalizing — slug: <slug>
[finalize]   solution → <absolute-path-to-solution-dir>
```

---

## What to do

Read `slug`, `project`, `repo_root`, `review_iteration`, `winning_agent` from STATE.json. The solution is in `~/.claude/runs/implement/<slug>/solution/<project>/`.

### 1. Rename branch and push

Determine the final branch name:
- New feature → `feature/<slug>`
- Bug fix → `fix/<slug>`

```bash
git -C <solution-path> branch -m solution/<slug> <final-branch>
```

Print `[git] ├ START  creating branch <final-branch>`

```bash
git -C <solution-path> push -u origin <final-branch>
```

Print `[git] ├ DONE   pushed → <final-branch>`

### 2. Open draft PR

Using `gh pr create --draft` with:
- Title: derived from the Conventional Commits message (e.g. `feat: add pagination to users API`)
- Body: summarize the change, link the originating ticket if one exists, list review iterations completed

Print:
```
[gh] ├ START  opening draft PR
[gh] ├ DONE   <full-pr-url>
```

### 3. Update STATE.json

Set `"step": "finalized"`, `"branch": "<final-branch>"`, `"pr_url": "<pr-url>"`.

### 4. Print the PR URL (machine-readable)

Print the PR URL on its own line — the autofix workflow parses this:
```
<full-pr-url>
```

### 5. Print the implementation summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Implementation Summary — <slug>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## Agents  →  <absolute-drafts-dir>

agent-1  <1–4 sentences: approach, files changed, CI result>
         <absolute-path-to-agent-1-worktree>

agent-2  ...

## Selection  →  <absolute-path-to-SELECTION_REASONING.md>

Chose <winning_agent>. <1–4 sentences: why it won>

## Review  →  <absolute-path-to-reviews-dir>

Iteration 1 — CHANGES REQUESTED
  Finding: <1-sentence summary>  →  <path-to-review.md>
  Fix:     <1-sentence summary>  →  <path-to-changes.md>

Iteration 2 — APPROVED
  report  →  <path-to-review.md>

## Final Solution  →  <absolute-path-to-solution-worktree>

<3–6 sentences: what files changed, core logic, how it's tested, anything notable>

PR  →  <full-pr-url>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Print `[finalize] └ DONE`

### 6. Write GitHub step summary

If `$GITHUB_STEP_SUMMARY` is set, append the full implementation summary (same content as above, formatted as markdown).

---

## Notes
- The PR URL must appear as a standalone line in stdout so the autofix workflow can parse it with `grep -oE 'https://github\.com/...'`
- Do NOT remove the solution worktree — leave it in place for inspection
