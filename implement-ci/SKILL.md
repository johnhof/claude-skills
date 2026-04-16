# implement-ci

Stage 6 of the split implement workflow. Waits for CI to pass on the created PR. If checks fail, reads the failure logs, fixes the code, commits, pushes, and waits again. Repeats until CI passes or max iterations are reached.

Prior stage: `implement-finalize`

---

## Resume / state check

Read `~/.claude/runs/implement/.current` to get the active slug. Then read STATE.json.

- If `step` is `finalized` and `pr_url` is set: proceed
- If `step` is `ci_passed`: already done — print status and exit
- If `step` is `ci_failed_max`: print the unresolved failures and exit 1
- Otherwise: print an error — wrong stage — and exit

Print:
```
[ci] ┌ Watching CI — slug: <slug>
[ci]   PR → <pr_url>
[ci]   branch → <branch>
```

---

## What to do

Read `pr_url`, `branch`, `repo_root`, `project` from STATE.json. The solution worktree is at `~/.claude/runs/implement/<slug>/solution/<project>/`.

Max fix iterations: **5**

### Poll loop

For each iteration:

#### 1. Wait for checks to complete

Run:
```bash
gh pr checks <pr_url> --watch --fail-fast 2>&1
```

This streams check status to the log in real time and exits when all checks complete (exit 0 = all passing, exit 1 = one or more failed).

If all checks pass:
- Update STATE.json: `"step": "ci_passed"`
- Print `[ci] └ DONE   CI passed ✓`
- Exit

#### 2. If checks failed — diagnose

Print `[ci] ├ FAIL   checks failed — iteration N of 5`

Fetch failure details:
```bash
# Get the failed check run IDs
gh pr checks <pr_url> --json name,conclusion,detailsUrl | jq '.[] | select(.conclusion == "failure")'

# Get logs for each failed run
gh run view <run_id> --repo <repo> --log-failed
```

Print a summary of what failed:
```
[ci] ├ failed checks:
[ci] ├   - <check-name>: <first line of failure>
```

#### 3. Fix the failures

Print `[ci] ├ fixing — reading failure logs and patching code`

Launch a fix agent in the solution worktree (`~/.claude/runs/implement/<slug>/solution/<project>/`) that:
- Reads the full failure logs
- Identifies the root cause (test failure, lint error, type error, build error, etc.)
- Makes the minimum code change needed to fix it
- Does NOT refactor or change anything unrelated to the failure
- Runs the failing check locally to confirm the fix before committing (e.g. `npm test`, `npm run lint`, `npx tsc --noEmit`)
- If the local run still fails, iterates on the fix until it passes locally

#### 4. Commit and push

In the solution worktree:
```bash
git add -A
git commit -m "fix: address CI failures (<brief description of what was fixed>)"
git push
```

Print:
```
[ci] ├ pushed fix — waiting for CI to re-run
```

Update STATE.json: `"ci_fix_iteration": N`

Return to step 1 (wait for checks again).

### If max iterations reached

Update STATE.json: `"step": "ci_failed_max"`

Post a comment on the PR:
```bash
gh pr comment <pr_url> --body "<comment>"
```

Comment format:
```markdown
<!-- autofix-bot -->
## ⚠️ CI still failing after 5 fix attempts

Autofix was unable to get CI green. Here are the unresolved failures:

| Check | Failure |
|-------|---------|
| <check-name> | <brief summary of error> |
| <check-name> | <brief summary of error> |

<details>
<summary>Last failure logs</summary>

\`\`\`
<truncated failure log, max ~50 lines>
\`\`\`
</details>

Please review and fix manually.
```

Print:
```
[ci] └ STOPPED — CI still failing after 5 fix attempts
[ci]   unresolved failures:
[ci]   - <check-name>: <summary>
[ci]   comment posted on PR
```

Print `CI_STATUS: FAILED` (machine-readable, for the autofix workflow)

Exit 1.

---

## GitHub step summary

If `$GITHUB_STEP_SUMMARY` is set, append after each iteration:

```markdown
## CI Watch

| Iteration | Status | Fixed |
|-----------|--------|-------|
| 1 | ❌ FAIL | lint: unused import in foo.ts |
| 2 | ✅ PASS | — |
```

---

## Notes
- Use `gh pr checks --watch` for real-time streaming — do not poll manually with sleep loops
- The fix agent must run the failing check locally before committing to avoid push-fix-fail cycles
- Only fix what CI is complaining about — do not touch passing checks or unrelated code
- If the same check keeps failing after 2 iterations with different errors, read the full log carefully before attempting a third fix — the root cause may be deeper than it appears
