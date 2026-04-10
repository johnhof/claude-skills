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
  comments.json        ← raw fetched comments (top-level + inline)
  plan.md              ← resolution plan from Step 3
  responses.md         ← log of all comments posted / reactions added
  ci.md                ← CI failure details and fix attempts
  CYCLE_1.md           ← full terminal output for processing cycle 1
  CYCLE_2.md           ← full terminal output for processing cycle 2 (if new events triggered a re-run)
  ...
```

**Logging rules:**
- Every line printed to the terminal during a cycle MUST also be appended to the current `CYCLE_#.md`
- Cycle 1 begins at Step 1. Each time the poll loop (Step 8) detects new actionable events and re-enters Step 2, increment the cycle counter and start writing to a new `CYCLE_#.md`
- Timestamps should be prepended to each logged line in `CYCLE_#.md`: `[HH:MM:SS] <line>`

---

## Step 1 — Fetch PR Details

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

For each comment (top-level and inline), evaluate:

- **Valid & actionable** — the comment identifies a real issue that can be fixed with a code change
- **Valid but not actionable** — the comment is a question, acknowledgement, or discussion that doesn't require a code change
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
- **Invalid/not-actionable comments** — plan a response explaining the agent's reasoning

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

[INVALID — will respond]
  Comment: @dave line 88
  Reason: The method is called in <file>:<line> — this is not dead code.

[NOT ACTIONABLE — will respond]
  Comment: @eve top-level
  Reason: This is an acknowledgement, no code change needed.
```

---

## Step 4 — Post Initial Responses

**Without waiting for user input**, post responses to all invalid and not-actionable comments immediately, before making any code changes.

For each **invalid or not-actionable** comment, post a reply via:
```bash
gh api /repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies \
  -f body="<explanation>

---
🤖 PR Helper"
```

For top-level comments use:
```bash
gh api /repos/{owner}/{repo}/issues/{number}/comments \
  -f body="<explanation>

---
🤖 PR Helper"
```

For each **valid** comment, add a 👍 emoji reaction via the GitHub reactions API:
```bash
gh api /repos/{owner}/{repo}/pulls/comments/{comment_id}/reactions \
  -X POST -f content="+1"
```

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
6. Post a reply to each comment addressed by this commit:
   ```
   Addressed in <owner>/<repo>/commit/<sha> — <one-line description of what changed>.

   ---
   🤖 PR Helper
   ```
   Use the same `gh api` endpoints as Step 4. Include the full GitHub commit URL so the commenter can review the isolated change.

**Do not resolve or dismiss any comments.** Leave resolution to humans.

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
4. If not locally reproducible, push the best-effort fix and note it in a top-level PR comment

---

## Step 7 — Other Blockers

Check for remaining blockers:
- **Merge conflicts** — attempt to rebase or merge the base branch and resolve conflicts; commit and push
- **Required reviews not yet approved** — post a top-level comment tagging the required reviewers that the PR is ready for re-review (do not attempt to bypass)
- **Branch protection / status checks still pending** — note them but do not attempt to bypass

---

## Step 8 — Poll Until Merged

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
- **New comments** since last check → increment cycle counter, open new `CYCLE_#.md`, loop back to Step 2 for new comments only
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

- **Never pause for user input** after Step 0 — the workflow runs fully autonomously from Step 1 through Step 8
- Never resolve/dismiss comments — leave that to humans
- Every automated comment and response MUST end with `\n\n---\n🤖 PR Helper`
- Always work on the PR's head branch; never commit directly to base
- If the head branch is not checked out locally, run `gh pr checkout <number>` first
- When in doubt about whether a comment is valid, treat it as valid and fix it
- Keep each fix commit as small and focused as possible — one logical change per commit
