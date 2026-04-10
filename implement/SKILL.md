# Implementation Workflow

Multi-agent workflow for implementing complex code changes with parallel drafts, automated selection, iterative code review, and finalization.

## Resume Detection

Before starting any step, check whether a run directory for this prompt already exists and determine where to resume. Inspect the filesystem state and apply the first matching rule:

| Condition | Resume at |
|-----------|-----------|
| Implementations directory does not exist | Step 1 — full start |
| Implementations directory exists, all drafts are empty or absent | Step 1 — re-run agents |
| All 5 drafts contain implementations, but `solution/` is empty or absent | Step 2 — selection |
| `solution/` contains files, `reviews/` is empty or absent | Step 3 — iteration 1 |
| `reviews/iteration-N/review.md` exists and contains `APPROVED` | Step 4 — finalize |
| `reviews/iteration-N/review.md` exists but no `APPROVED` found, and no `iteration-N+1/` exists | Step 3 — iteration N+1 (continue review cycle) |
| `reviews/iteration-N/review.md` exists and `CHANGES REQUESTED`, but `reviews/iteration-N/changes.md` is absent | Step 3 — re-run fix agent for iteration N |

**How to determine "a draft contains an implementation"**: the draft directory is non-empty and contains at least one modified file relative to its base commit (i.e., `git -C <draft> diff HEAD --name-only` returns results, or the draft has commits beyond the branch point).

When resuming, print a clear status before proceeding:
```
[workflow] ⟳ Resuming from Step N — <reason detected from filesystem>
[workflow]   run dir → <absolute-path>
```

If the state is ambiguous (e.g., some drafts have work and some don't), print a warning and ask the user whether to restart the affected step or continue with what's available.

## Observability

Print a status line at the **start and completion of every step and sub-agent**. Use the format below. Directory paths must be printed as absolute paths so the terminal renders them as clickable links.

```
[workflow] ┌ Step 1/4 — Parallel Implementation
[agent-1]  ├ START  → ~/.claude/runs/implement/{{prompt-slug}}/drafts/agent-1/{{project}}
[agent-2]  ├ START  → ~/.claude/runs/implement/{{prompt-slug}}/drafts/agent-2/{{project}}
[agent-3]  ├ START  → ~/.claude/runs/implement/{{prompt-slug}}/drafts/agent-3/{{project}}
[agent-4]  ├ START  → ~/.claude/runs/implement/{{prompt-slug}}/drafts/agent-4/{{project}}
[agent-5]  ├ START  → ~/.claude/runs/implement/{{prompt-slug}}/drafts/agent-5/{{project}}
[agent-3]  ├ DONE   → ~/.claude/runs/implement/{{prompt-slug}}/drafts/agent-3/{{project}}
[agent-1]  ├ DONE   → ~/.claude/runs/implement/{{prompt-slug}}/drafts/agent-1/{{project}}
[agent-5]  ├ DONE   → ~/.claude/runs/implement/{{prompt-slug}}/drafts/agent-5/{{project}}
[agent-2]  ├ DONE   → ~/.claude/runs/implement/{{prompt-slug}}/drafts/agent-2/{{project}}
[agent-4]  ├ DONE   → ~/.claude/runs/implement/{{prompt-slug}}/drafts/agent-4/{{project}}
[workflow] └ DONE   Step 1/4 — all 5 agents complete

[workflow] ┌ Step 2/4 — Selection
[selector] ├ START  comparing 5 implementations
[selector] ├ DONE   selected agent-3 — clearest separation of concerns, smallest diff
[selector] ├        rejected: agent-1 (redundant abstraction), agent-2 (missing error handling),
[selector] ├                  agent-4 (inconsistent naming), agent-5 (test coverage gaps)
[selector] ├        solution → ~/.claude/runs/implement/{{prompt-slug}}/solution
[workflow] └ DONE   Step 2/4

[workflow] ┌ Step 3/4 — Review iteration 1 of 5
[sync]     ├ START  merging origin/main
[sync]     ├ DONE   no conflicts
[reviewer] ├ START  scanning diff + codebase context
[reviewer] ├ DONE   CHANGES REQUESTED — 2 Blockers, 1 Major
[reviewer] ├        report → ~/.claude/runs/implement/{{prompt-slug}}/reviews/iteration-1/review.md
[fixer]    ├ START  addressing 2 Blockers, 1 Major
[fixer]    ├ DONE   changes → ~/.claude/runs/implement/{{prompt-slug}}/reviews/iteration-1/changes.md
[workflow] └ DONE   Step 3/4 — iteration 1 complete

[workflow] ┌ Step 3/4 — Review iteration 2 of 5
[sync]     ├ START  merging origin/main
[sync]     ├ DONE   no conflicts
[reviewer] ├ START  scanning diff + codebase context
[reviewer] ├ DONE   APPROVED
[reviewer] ├        report → ~/.claude/runs/implement/{{prompt-slug}}/reviews/iteration-2/review.md
[workflow] └ DONE   Step 3/4 — APPROVED after 2 iterations

[workflow] ┌ Step 4/4 — Finalize
[git]      ├ START  creating branch feature/{{prompt-slug}}
[git]      ├ DONE   committed — "feat: {{summary}}"
[git]      ├ START  pushing to remote
[git]      ├ DONE   pushed
[gh]       ├ START  opening draft PR
[gh]       ├ DONE   https://github.com/{{org}}/{{project}}/pull/{{number}}
[workflow] └ DONE   Step 4/4
```

Every directory path printed must be the full absolute path (expand `~` to the actual home directory) so terminals render it as a clickable file link.

## Directory Structure

Every prompt execution creates a run directory at:
```
~/.claude/runs/implement/{{prompt-slug}}/
```

Where `{{prompt-slug}}` is a short kebab-case summary of the prompt (e.g., `add-user-auth-endpoint`), guaranteed non-conflicting — append `-2`, `-3`, etc. if the slug already exists.

Full layout:
```
~/.claude/runs/implement/{{prompt-slug}}/
  README.md                              ← full text of the optimized prompt
  SELECTION_REASONING.md                 ← per-agent summaries and selection rationale
  drafts/
    agent-1/{{project}}/                 ← agent 1's isolated git draft
    agent-2/{{project}}/                 ← agent 2's isolated git draft
    agent-3/{{project}}/
    agent-4/{{project}}/
    agent-5/{{project}}/
  solution/                              ← copy of the winning draft after selection
  reviews/
    iteration-1/
      review.md                          ← structured reviewer findings
      changes.md                         ← summary of fixes applied to address review
    iteration-2/
      review.md
      changes.md
    ...
```

Where `{{project}}` is the basename of the git root of the repo being modified.

Before launching agents, write the full optimized prompt text to `README.md` in the prompt slug directory.

## Step 1 — Parallel Implementation (5 Worktrees)

Print `[workflow] ┌ Step 1/4 — Parallel Implementation` before starting.

Launch **5 sub-agents in parallel**, each in an isolated git draft stored at:
```
/Users/{{user}}/.claude/implementations/{{prompt-slug}}/drafts/agent-N/{{project}}/
```

Each agent implements the approved prompt independently and may diverge in approach — that is intentional. Give each agent the full optimized prompt plus the instruction to implement it completely, with the following non-negotiable constraints:

- **Minimal invasiveness**: touch only the files and lines required to satisfy the prompt. Do not refactor, reformat, or "clean up" surrounding code that isn't directly part of the change.
- **Surgical diffs**: the diff should be as small as possible. Every line changed must be justified by a direct requirement of the task.
- **Simple design**: prefer the simplest solution that correctly satisfies the requirements. Avoid new abstractions, helpers, or patterns unless the task explicitly demands them. When in doubt, inline it.

Each agent must follow **red/green TDD** — tests are written before implementation code:

1. **Red** — write all tests for the new behavior first. Run them and confirm they fail (if they pass before implementation exists, the tests are wrong — fix them). Do not write any implementation code until tests are failing for the right reason.
2. **Green** — write the minimum implementation code needed to make the failing tests pass. Run tests again and confirm they are all green.
3. **Refactor** — clean up the implementation (not the tests) without changing behavior. Re-run tests to confirm still green.

Each agent must, after completing the red/green/refactor cycle:
1. **Detect CI steps** — inspect the repo for CI configuration (`.github/workflows/`, `Makefile`, `package.json` scripts, `Taskfile`, `circle.yml`, etc.) and identify build, lint, and test steps
2. **Run all detected CI steps** in the draft — build first, then lint, then tests
3. **Enforce 80% coverage** — run coverage and confirm new code reaches at least 80% line/branch coverage; if coverage falls below 80%, write additional tests until it passes
4. **Record CI results** — note which steps passed/failed and the final coverage percentage; this becomes part of the agent's output used during selection

For each agent:
- Print `[agent-N] ├ START  → <absolute-draft-path>` immediately when the agent is launched
- Print `[agent-N] ├ red    tests written — N failing` after the red step
- Print `[agent-N] ├ green  tests passing — N passed` after the green step
- Print `[agent-N] ├ ci     build: PASS | lint: PASS | tests: PASS (coverage: 84%)` (or FAIL with reason) as CI results come in
- Print `[agent-N] ├ DONE   → <absolute-draft-path>` when the agent finishes

If a CI step fails and cannot be fixed, the agent must note it prominently — the selection agent will penalize implementations with CI failures.

Print `[workflow] └ DONE   Step 1/4 — all 5 agents complete` once all agents have finished.

## Step 2 — Selection Agent

Print `[workflow] ┌ Step 2/4 — Selection` before starting.
Print `[selector] ├ START  comparing 5 implementations` when the selection agent launches.

Launch a **selection agent** that:
- Reviews all 5 implementations side-by-side
- Evaluates each against: correctness, code clarity, DRY adherence, implementation consistency, test coverage, minimal diff size, **and CI results** (a passing build/tests with ≥80% coverage is a hard requirement — implementations that failed CI and couldn't self-correct are eliminated first)
- **Requires TDD discipline** — eliminates any implementation where tests were not written before implementation code (i.e., no evidence of a red phase, or tests that couldn't have failed before the implementation existed)
- **Strongly prefers** the implementation with the smallest, most focused diff — an implementation that touches fewer files and introduces no unnecessary abstractions scores higher, all else being equal
- **Penalizes** any implementation that refactors or reformats code outside the required scope, introduces new helper abstractions not demanded by the task, or over-engineers a simple problem
- Picks the single best implementation and explains why the others were rejected
- Writes `~/.claude/runs/implement/{{prompt-slug}}/SELECTION_REASONING.md` containing:
  - A **summary of each agent's implementation** — what approach it took, which files it changed, and notable design decisions
  - The **selection decision** — which agent was chosen and the specific reasons it won
  - The **rejection reasons** for each losing agent — concrete, per-agent explanations referencing diff size, design choices, CI results, TDD adherence, etc.
- Copies the winning draft contents into the absolute path of `~/.claude/runs/implement/{{prompt-slug}}/solution/`
- **Never deletes any drafts** — all 5 drafts are preserved in `drafts/agent-N/` for later analysis

When the selection agent completes, print:
```
[selector] ├ DONE   selected agent-N — <brief reason>
[selector] ├        rejected: agent-X (<reason>), agent-X (<reason>), ...
[selector] ├        reasoning → <absolute-path-to-SELECTION_REASONING.md>
[selector] ├        solution → <absolute-path-to-solution-dir>
[workflow] └ DONE   Step 2/4
```

## Step 3 — Code Review Cycle

For each iteration, print `[workflow] ┌ Step 3/4 — Review iteration N of 5` before starting.

Using the solution directory, enter a **review-fix loop** (max 5 iterations):

### Pre-Review Main Sync

Print `[sync] ├ START  merging origin/main` before syncing.

In the solution draft:
1. Run `git fetch origin && git merge origin/main`
2. If merge conflicts exist, resolve them automatically:
   - Files touched by the implementation: prefer the implementation's version; re-apply unrelated semantic changes from main
   - Files not touched by the implementation: prefer main
   - If resolution is ambiguous: prefer main, print `[sync] ├ WARNING ambiguous conflict in <file> — preferred main`
3. Print `[sync] ├ DONE   no conflicts` or `[sync] ├ DONE   N conflicts resolved`

### Review

Print `[reviewer] ├ START  scanning diff + codebase context` when the reviewer agent launches.

Launch a **reviewer agent** that:
- Reviews **only the changed files and lines introduced by this implementation** — not pre-existing code
- Scans the broader codebase for context only (DRY violations, consistency with existing patterns, integration points), but **never raises findings about pre-existing code not touched by this change**
- Uses `~/.claude/CODE_REVIEW.md` as its rubric, scoped strictly to what was added, modified, or deleted in this diff
- Every finding must cite the specific file and line number from the diff — findings without a direct diff reference are invalid and must be omitted
- **Runs all CI steps** (build, lint, tests, coverage) against the solution and includes the results in the report header; any CI failure is an automatic Blocker; coverage on new code below 80% is an automatic Major
- Writes the full structured report (CI results → Blockers → Majors → Minors → Nits) to the absolute path of `~/.claude/runs/implement/{{prompt-slug}}/reviews/iteration-N/review.md`
- Declares either `APPROVED` or `CHANGES REQUESTED`

When the reviewer completes, print:
```
[reviewer] ├ DONE   APPROVED
[reviewer] ├        report → <absolute-path-to-review.md>
[workflow] └ DONE   Step 3/4 — APPROVED after N iterations
```
or:
```
[reviewer] ├ DONE   CHANGES REQUESTED — N Blockers, N Majors, N Minors
[reviewer] ├        report → <absolute-path-to-review.md>
```

### If `CHANGES REQUESTED`:

Print `[fixer] ├ START  addressing N Blockers, N Majors` when the fix agent launches.

- Launch a **fix agent** to implement all Blocker and Major findings
- After applying fixes, the fix agent must re-run all CI steps (build, lint, tests, coverage) in the solution draft
- If any CI step fails after fixes, the fix agent must resolve the failure before completing
- Coverage must remain at or above 80% on new code — if fixes reduced coverage, write additional tests
- Write a summary of every fix applied (including any CI remediation) to the absolute path of `~/.claude/runs/implement/{{prompt-slug}}/reviews/iteration-N/changes.md`

When the fix agent completes, print:
```
[fixer]    ├ ci     build: PASS | lint: PASS | tests: PASS (coverage: 84%)
[fixer]    ├ DONE   changes → <absolute-path-to-changes.md>
[workflow] └ DONE   Step 3/4 — iteration N complete, starting iteration N+1
```

Increment the iteration counter and return to Pre-Review Main Sync.

### If `APPROVED` or iteration count reaches 5:
- If approved: proceed to Step 4
- If max iterations hit: print `[workflow] └ STOPPED — max review iterations reached` followed by the list of unresolved findings, and ask the user to review manually

## Step 4 — Finalize: Branch and Apply

Print `[workflow] ┌ Step 4/4 — Finalize` before starting.

1. In the original repo, create a branch:
   - `feature/{{prompt-slug}}` for new features
   - `fix/{{prompt-slug}}` for bug fixes
2. Copy all changed files from `solution/` into the repo on that branch
3. Stage and commit using a Conventional Commits message summarizing the change

   Print:
   ```
   [git]      ├ START  creating branch <branch-name>
   [git]      ├ DONE   committed — "<conventional-commit-message>"
   ```

4. Push the branch to remote: `git push -u origin <branch>`

   Print:
   ```
   [git]      ├ START  pushing to remote
   [git]      ├ DONE   pushed → <branch-name>
   ```

5. Open a **draft PR** using `gh pr create --draft` with:
   - Title derived from the Conventional Commits message
   - Body summarizing the change, linking to the originating ticket if one was provided, and listing the review iterations completed

   Print:
   ```
   [gh]       ├ START  opening draft PR
   [gh]       ├ DONE   <full-pr-url>
   ```

6. Run `/arbiter` to generate a review link and present it to the user

   Print:
   ```
   [workflow] └ DONE   Step 4/4
   ```

## Notes
- Work artifacts for every execution are preserved at `~/.claude/runs/implement/` for inspection
- Review history (findings + applied fixes per iteration) lives at `reviews/iteration-N/`
- All 5 agent drafts are always preserved in `drafts/agent-N/` — never delete them; clean up `~/.claude/runs/implement/` manually when done
- Each review iteration should be stricter than the last — do not re-raise issues already resolved
- The fix agent must not introduce new issues while resolving existing ones
- The reviewer agent must always reference `~/.claude/CODE_REVIEW.md` — never improvise review criteria
- **Scope discipline**: the reviewer must never flag issues in code that predates this change. If a pre-existing problem is noticed, it may be mentioned once as an informational note (not a finding), and only if it directly interacts with the changed code
