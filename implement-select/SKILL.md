# implement-select

Stage 3 of the split implement workflow. Compares all agent drafts and selects the best one, creating a solution worktree from the winner.

Prior stage: `implement-draft` | Next stage: `implement-review`

---

## Resume / state check

Read `~/.claude/runs/implement/.current` to get the active slug. Then read STATE.json.

- If `step` is `draft_done`: proceed normally
- If `step` is `select_done`: already selected — print which agent won and exit (run `implement-review` next)
- Otherwise: print an error — wrong stage — and exit

---

## What to do

Read `agent_count`, `project`, `repo_root`, `slug` from STATE.json.

### Run the selection agent

Print:
```
[select] ┌ Comparing N implementations
[select] ├ START
```

The selection agent:

1. Reviews all N drafts side-by-side — diffs, test files, CI results
2. Applies these criteria (in priority order):
   - **CI hard requirement**: any draft that failed CI and couldn't self-correct is eliminated immediately
   - **TDD discipline**: any draft with no evidence of a red phase (tests written before implementation) is disqualified
   - **Correctness**: does it fully satisfy the SPEC?
   - **Minimal diff**: smallest focused change wins, all else equal
   - **Code clarity and consistency** with existing patterns
   - **No unnecessary abstractions** — penalize over-engineering
3. Picks the single best draft
4. Writes `~/.claude/runs/implement/<slug>/SELECTION_REASONING.md` containing:
   - Summary of each agent's approach, files changed, and CI result
   - Divergence analysis (structural, logical decisions, test strategy, convergence verdict)
   - Selection decision with specific reasons
   - Rejection reasons per losing agent (concrete, referencing diff size, CI, TDD, etc.)

### Create the solution worktree

From the winning draft's HEAD SHA, create the solution branch and worktree:

```bash
SHA=$(git -C <winning-draft-path> rev-parse HEAD)
git -C <repo_root> branch solution/<slug> <SHA>
git -C <repo_root> worktree add ~/.claude/runs/implement/<slug>/solution/<project> solution/<slug>
```

Print:
```
[select] ├ DONE   selected agent-N — <brief reason>
[select] ├        rejected: agent-X (<reason>), ...
[select] ├        reasoning → <absolute-path-to-SELECTION_REASONING.md>
[select] ├        solution  → <absolute-path-to-solution-dir>
[select] └ DONE
```

### Update STATE.json

Set `"step": "select_done"` and `"winning_agent": "agent-N"`.

### Write GitHub step summary

If `$GITHUB_STEP_SUMMARY` is set, append:

```markdown
## Selection

**Winner:** agent-N — <brief reason>

**Rejected:**
- agent-X: <reason>
- agent-Y: <reason>

**Divergence:** <convergence verdict — e.g. "agents converged on nearly identical approaches" or "3 distinct strategies surfaced">

[Full reasoning](<absolute-path-to-SELECTION_REASONING.md>)
```

---

## Notes
- Never delete any draft worktrees — all N are preserved for inspection
- The selection reasoning is the permanent record of why the winner was chosen; be specific
- If all drafts failed CI, pick the least broken one and flag it clearly in STATE.json with `"ci_warning": true`
