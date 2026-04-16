# implement-draft

Stage 2 of the split implement workflow. Runs AGENT_COUNT parallel implementation agents, each working in an isolated git worktree.

Prior stage: `implement-plan` | Next stage: `implement-select`

---

## Resume / state check

Read `~/.claude/runs/implement/.current` to get the active slug. Then read `~/.claude/runs/implement/<slug>/STATE.json`.

- If `step` is `plan_done`: proceed normally — start Step 1
- If `step` is `draft_done`: all agents already finished — print status and exit (nothing to do; run `implement-select` next)
- If `step` is anything that indicates drafts are partially complete (some agent dirs exist but others don't): print a warning listing which agents are done and which are missing, then re-run only the missing agents
- If `step` is anything else: print an error — wrong stage — and exit

Print:
```
[draft] ⟳ Resuming — step: <step>, slug: <slug>
[draft]   run dir → /Users/<user>/.claude/runs/implement/<slug>
```
or:
```
[draft] ┌ Starting — slug: <slug>, agents: N
[draft]   run dir → /Users/<user>/.claude/runs/implement/<slug>
```

---

## What to do

Read `agent_count`, `project`, `repo_root`, and `prompt` from STATE.json. Read the SPEC from `~/.claude/runs/implement/<slug>/SPEC.md`.

### Launch AGENT_COUNT parallel implementation agents

Each agent works in an isolated git worktree at:
```
~/.claude/runs/implement/<slug>/drafts/agent-N/<project>/
```

To create an agent's worktree:
1. `git -C <repo_root> worktree add ~/.claude/runs/implement/<slug>/drafts/agent-N/<project> HEAD`

Print `[agent-N] ├ START  → <absolute-draft-path>` when each agent launches.

Each agent receives the full SPEC and must:
- Implement the change completely and independently
- Follow **red/green TDD** (write failing tests first, then implement, then refactor)
  - Bug fix: write a reproduction test that fails against current code first
  - Feature: write all tests before any implementation code
- After TDD cycle:
  1. **Detect CI steps** — check `.github/workflows/`, `Makefile`, `package.json`, `Taskfile`, etc.
  2. **Run build, lint, tests** in the worktree
  3. **Enforce 80% coverage** on new code — write more tests if below
  4. **Record CI results** (pass/fail + coverage %)
- Constraints: minimal diff, surgical changes only, no refactoring surrounding code, no speculative abstractions

Agent status lines:
```
[agent-N] ├ red    tests written — N failing
[agent-N] ├ green  tests passing — N passed
[agent-N] ├ ci     build: PASS | lint: PASS | tests: PASS (coverage: 84%)
[agent-N] ├ DONE   → <absolute-draft-path>
```

Print `[draft] └ DONE   all N agents complete` when all finish.

### Clarification check

After all agents complete, check if any agent output contained `NEEDS_CLARIFICATION`. If so:
- Print `NEEDS_CLARIFICATION` (machine-readable, for autofix workflow)
- Update STATE.json: `"step": "needs_clarification"`
- Exit

### Update STATE.json

Set `"step": "draft_done"`.

### Write GitHub step summary

If `$GITHUB_STEP_SUMMARY` is set, append:

```markdown
## Draft — N Agents

| Agent | TDD | CI | Coverage | Notes |
|-------|-----|-----|----------|-------|
| agent-1 | ✓ | PASS | 84% | <brief approach> |
| agent-2 | ✓ | PASS | 91% | <brief approach> |
| agent-3 | ✓ | FAIL | — | <failure reason> |
```

---

## Notes
- All AGENT_COUNT agents run in parallel — launch them concurrently
- Never delete a draft worktree — all are preserved for the selection stage
- If a CI step fails and the agent can't fix it, note it prominently; the selector will penalize it
- The TDD red phase is mandatory — any agent that writes implementation before tests is automatically disqualified in selection
