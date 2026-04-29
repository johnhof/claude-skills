---
name: pr-helper
description: Automatically address PR comments, CI failures, and blocking issues. Use when the user says "address PR comments", "fix PR", "handle PR feedback", "resolve PR issues", or similar. Takes a PR URL or number as a parameter; if none provided, infers from context.
---

# PR Helper

Automatically triage and address all comments, CI failures, and blockers on a pull request, then poll until it merges.

**The only point where user interaction is required is the initial PR confirmation in Step 0 when no PR was explicitly provided. Every other step executes immediately without waiting.**

---

## Step 0 — Determine the PR

If the user provided a PR URL or number, use it directly and skip to Step 1.

Otherwise, scan the conversation history for:
- Recently mentioned branch names (e.g. `feature/...`, `fix/...`)
- Recently pushed commits or `gh pr create` output
- Any PR URLs mentioned in prior messages

Identify the most likely PR. Before proceeding, ask the user:

> I believe you're referring to **[PR title](url)** — is that correct?

Wait for confirmation. Once confirmed, proceed.

---

## Run Directory

Before doing anything else, check for an existing run directory at:
```
~/.claude/runs/pr-helper/<repo>-pr-<number>/
```

If it exists:
- Read `README.md` for prior PR context
- Read `sticky_comment_id` for the ID of the existing top-level comment
- Count existing `CYCLE_#.md` files to determine the next cycle number
- Resume from where the last cycle left off — skip re-writing `README.md`, proceed directly to Step 1 to re-fetch the latest PR state, and open the next `CYCLE_#.md`
- Print: `[pr-helper] resuming existing run — cycle N starting`

If it does not exist, create it and start fresh.

Every pr-helper execution uses a run directory at:
```
~/.claude/runs/pr-helper/<repo>-pr-<number>/
```

Full layout:
```
~/.claude/runs/pr-helper/<repo>-pr-<number>/
  README.md            ← PR summary written at start of Step 1
  sticky_comment_id    ← GitHub comment ID of the single top-level sticky comment
  comments.json        ← raw fetched comments (top-level + inline)
  plan.md              ← resolution plan from Step 3
  ci.md                ← CI failure details and fix attempts
  CYCLE_1.md           ← full terminal output for processing cycle 1
  CYCLE_2.md           ← full terminal output for processing cycle 2 (if new events triggered a re-run)
  ...
```

**Logging rules:**
- Every line printed to the terminal during a cycle MUST also be appended to the current `CYCLE_#.md`
- Cycle 1 begins at Step 1. Each time the poll loop (Step 9) detects new actionable events and re-enters Step 2, increment the cycle counter and start writing to a new `CYCLE_#.md`
- Timestamps should be prepended to each logged line in `CYCLE_#.md`: `[HH:MM:SS] <line>`

---

## Step 1 — Sync Branch

**At the start of every cycle (including cycle 1)**, before fetching PR details, pull the latest base branch and rebase the head branch onto it:

```bash
git fetch origin
git rebase origin/<base-branch>
```

If conflicts arise during the rebase, resolve them immediately (same process as Step 7 merge conflicts: resolve markers, `git add`, `git rebase --continue`). Push the rebased branch:

```bash
git push --force-with-lease origin <head-branch>
```

Do not proceed to the next step until the branch is cleanly rebased with no conflicts.

---

## Step 2 — Fetch PR Details

Using `gh pr view <url-or-number> --json number,title,body,baseRefName,headRefName,url,state,reviews,reviewRequests,statusCheckRollup,comments` and `gh api /repos/{owner}/{repo}/pulls/{number}/comments` (inline comments), compile:

- PR title, description, base/head branches
- All **top-level conversation comments** (author, body, timestamp)
- All **file/line-specific review comments** (author, file, line, body, timestamp)
- CI status (passing / failing / pending, with job names and failure details)
- Any other blockers: merge conflicts, required reviews not approved, branch protection rules

**Write `README.md`** to the run directory immediately after fetching, before printing anything else (skip if resuming an existing run — `README.md` already exists):

```markdown
# PR #<number> — <title>

**URL:** <url>
**Branch:** <head> → <base>
**Author:** <author>
**State:** <state>
**Opened:** <created_at>

## Description

<pr body>

## Changeset

<output of: git diff <base>...<head> --stat>
```

**Create or verify the sticky comment.** There must be exactly one top-level PR Helper comment for the lifetime of this PR. On the very first run (no `sticky_comment_id` file), post the skeleton frame and save the comment ID:

```bash
STICKY_ID=$(gh api /repos/{owner}/{repo}/issues/{number}/comments \
  -f body="## PR Helper

_Run in progress — updating as fixes are applied…_

| # | Reviewer | Comment | Resolution |
|---|----------|---------|------------|

### Commits
_none yet_

---
🤖 PR Helper" --jq '.id')
echo "$STICKY_ID" > ~/.claude/runs/pr-helper/<repo>-pr-<number>/sticky_comment_id
```

On resuming runs, read the saved ID — do **not** create a new comment.

**Open `CYCLE_1.md`** — all terminal output from this point forward is appended to it with `[HH:MM:SS]` timestamps until the cycle ends.

Print a formatted summary to the user:

```
━━━ PR #N — <title> ━━━
Branch:   <head> → <base>
State:    <open/draft>
CI:       <✅ passing | ❌ N jobs failing | ⏳ pending>

── Top-level comments (<N>) ──
  [@author] "<body preview>"
  ...

── Inline comments (<N>) ──
  [@author] <file>:<line> — "<body preview>"
  ...

── Other blockers ──
  <merge conflicts / required approvals / etc., or "none">
```

Do not wait for user interaction after printing this.

---

## Step 2 — Evaluate Comments

**First, filter out all non-review comments.** Only human code review comments are in scope. Silently discard — do not log, table, or react to — any comment that is:

- Posted by a bot account (author login ends in `[bot]`, or is a known bot: `github-actions`, `cloudflare-workers-and-pages`, `linear`, `greptile-apps`, `dependabot`, etc.)
- A slash command trigger (e.g. `@claude /review`, `/approve`, `/deploy`)
- A CI/deploy status notification or linkback (Cloudflare Pages preview URLs, Linear issue links, GitHub Actions summaries)
- An automated status update with no code feedback

Only evaluate comments that are **human-authored code review feedback** — questions, suggestions, bug reports, or change requests about the code in the diff.

For each qualifying comment, evaluate:

- **Valid & actionable** — the comment identifies a real issue that can be fixed with a code change
- **Valid but not actionable** — the comment is a question or discussion that doesn't require a code change
- **Invalid / incorrect** — the agent believes the comment is factually wrong, based on a misunderstanding of the code, or already addressed

Evaluation criteria:
- Does the comment reference something that actually exists in the current diff?
- Is the concern already addressed elsewhere?
- Is the suggestion consistent with the codebase's patterns and constraints?

---

## Step 3 — Build Resolution Plans

Group and plan fixes:

- **Independent comments** — plan each as a separate commit
- **Related comments** — plan as a single combined commit (note which comments are covered)
- **Invalid/not-actionable comments** — record the reasoning; it will appear in the sticky comment

For each plan, note:
- Which comment(s) it addresses
- What change will be made (or why no change is needed)
- Which files will be touched

Print the full plan to the user **without waiting for interaction**:

```
━━━ Resolution Plan ━━━

[WILL FIX — commit 1]
  Comments: @alice line 42, @bob line 44
  Plan: <description of change>
  Files: <file(s)>

[WILL FIX — commit 2]
  Comments: @charlie top-level
  Plan: <description of change>
  Files: <file(s)>

[INVALID — recorded in sticky comment]
  Comment: @dave line 88
  Reason: The method is called in <file>:<line> — this is not dead code.

[NOT ACTIONABLE — recorded in sticky comment]
  Comment: @eve top-level
  Reason: This is an acknowledgement, no code change needed.
```

---

## Step 4 — React to Valid Comments

For each **valid** comment, add a 👍 emoji reaction to acknowledge it will be addressed:
```bash
gh api /repos/{owner}/{repo}/pulls/comments/{comment_id}/reactions \
  -X POST -f content="+1"
```

**Do not post any comments — no replies, no thread responses, nothing.** All outcomes (fixed, invalid, not-actionable) are recorded only in the sticky comment updated in Step 8.

---

## Step 5 — Execute Fix Commits

Work on the checked-out head branch of the PR. For each fix plan:

1. Make the code changes
2. Stage only the files relevant to this fix
3. Commit with a message referencing the comment(s):
   ```
   fix: <description> (addresses @author comment)
   ```
4. Push the commit:
   ```bash
   git push origin <head-branch>
   ```
5. Get the commit SHA:
   ```bash
   git rev-parse HEAD
   ```

**Do not post any comments or replies after committing.** The sticky comment (Step 8) is the only place commit links and resolutions are recorded. Do not resolve or dismiss any comments — leave that to humans.

After all commits for this cycle are pushed, proceed to Step 6 (CI) then Step 7 (blockers), then run Step 8 to update the sticky comment and Step 8b to refresh the PR description.

---

## Step 6 — CI Failures

After all comment fixes are pushed, check CI status:
```bash
gh pr checks <number>
```

For each failing job:
1. Fetch the failure logs:
   ```bash
   gh run view <run-id> --log-failed
   ```
2. Attempt to reproduce the failure locally (run the same command that failed in CI)
3. If reproducible, fix the root cause, commit, and push:
   ```
   fix: resolve CI failure in <job-name>
   ```
4. If not locally reproducible, push the best-effort fix and note it in the sticky comment (Step 8) — do **not** post a new top-level comment.

---

## Step 7 — Other Blockers

### Merge Conflicts (REQUIRED — must be resolved before proceeding)

The start-of-cycle rebase in Step 1 handles conflicts proactively. If conflicts surface mid-cycle (e.g. a commit lands on base while fixes are being applied), resolve them using the same process as Step 1: fetch, rebase, resolve markers, `git add`, `git rebase --continue`, force-push. Do not proceed until `gh pr view <number> --json mergeable` returns `MERGEABLE`.

### Other Blockers

- **Required reviews not yet approved** — note the required reviewers in the sticky comment (Step 8); do not post a new top-level comment
- **Branch protection / status checks still pending** — note them in the sticky comment but do not attempt to bypass

---

## Step 8 — Update Sticky Comment

After all fixes are committed, CI is checked, and blockers are handled, **edit the existing sticky comment in-place** using its saved ID. Never post a new top-level comment — always PATCH the existing one:

```bash
STICKY_ID=$(cat ~/.claude/runs/pr-helper/<repo>-pr-<number>/sticky_comment_id)
gh api --method PATCH /repos/{owner}/{repo}/issues/comments/${STICKY_ID} \
  -f body="<updated body>"
```

The updated body must follow this format:

```
## PR Helper

**N comments addressed · N commits pushed · CI <✅ passing | ❌ N failing>**

### Comments

| # | Reviewer | Comment | Resolution |
|---|----------|---------|------------|
| 1 | @alice | <one-phrase summary of their comment> | Fixed in [`abc1234`](<commit-url>) — <one sentence what changed> |
| 2 | @bob | <one-phrase summary> | Not actionable — <one sentence explanation> |
| 3 | @charlie | <one-phrase summary> | Invalid — <one sentence explanation> |

### Commits
- [`abc1234`](<url>) fix: <message>
- [`def5678`](<url>) fix: <message>

### Notes
<any CI non-reproducible failures, required review blockers, or pending status checks — omit section if empty>

---
🤖 PR Helper
```

Rules:
- Every comment evaluated must appear in the table — fixed, invalid, and not-actionable alike
- The "Resolution" column must be a self-contained sentence
- Fixed comments must include the commit link; invalid/not-actionable must include the one-line reason
- Accumulate rows across cycles — do not reset the table on each cycle, append new rows

---

## Step 8b — Sync Branch (End of Cycle)

After updating the sticky comment, rebase the head branch onto the latest base branch again to incorporate any new commits that landed during this cycle:

```bash
git fetch origin
git rebase origin/<base-branch>
```

Resolve any conflicts as in Step 1. Push:

```bash
git push --force-with-lease origin <head-branch>
```

---

## Step 8c — Refresh PR Description

After every cycle (after Step 8b), read the current PR description and compare it to the actual state of the code on the head branch.

```bash
gh pr view <number> --json body --jq '.body'
```

Check whether the description accurately reflects:
- What the PR now does (after any fixes applied this cycle)
- The current approach taken (especially if a fix changed the implementation strategy)
- Any sections that reference code, behavior, or files that no longer exist as described

If the description is stale or incomplete, update it:
```bash
gh pr edit <number> --body "$(cat <<'EOF'
<updated description>
EOF
)"
```

Preserve any structured sections the author included (e.g. "## Summary", "## Test plan"). Only update content that is factually wrong or missing relative to the current code — do not rewrite for style.

---

## Step 9 — Poll Until Merged

Once all actionable items are addressed, enter a polling loop:

Every **10 seconds**, run:
```bash
gh pr view <number> --json state,mergedAt
```

Print on each poll:
```
[pr-helper] checking PR #N for actionable events... (state: open)
```

Check on each poll:
- **New comments** since last check → increment cycle counter, open new `CYCLE_#.md`, loop back to Step 1 (sync branch) then Step 3 for new comments only, then run Steps 8, 8b, and 8c to update the sticky comment, re-sync, and refresh the PR description
- **New CI failures** → increment cycle counter, open new `CYCLE_#.md`, loop back to Step 6
- **PR merged** → print the completion message and exit:

```
━━━ PR #N merged ━━━
<title> has been merged. All done! 🎉
```

- **PR closed without merge** → print:
```
━━━ PR #N closed ━━━
The PR was closed without merging. Exiting.
```

---

## Notes

- **Never pause for user input** after Step 0 — the workflow runs fully autonomously from Step 1 through Step 8b
- **The sticky comment is the only comment pr-helper ever posts or edits** — one comment, created in Step 1, updated in Step 8. No replies, no thread responses, no additional top-level comments under any circumstance
- Never resolve/dismiss comments — leave that to humans
- Every automated reply MUST end with `\n\n---\n🤖 PR Helper`
- Always work on the PR's head branch; never commit directly to base
- If the head branch is not checked out locally, run `gh pr checkout <number>` first
- When in doubt about whether a comment is valid, treat it as valid and fix it
- Keep each fix commit as small and focused as possible — one logical change per commit
