# implement-plan

Stage 1 of the split implement workflow. Analyzes the prompt, creates the run directory, writes SPEC.md and README.md, determines agent count, and initializes STATE.json.

Subsequent stages: `implement-draft` → `implement-select` → `implement-review` → `implement-finalize`

---

## Input

The skill argument (everything after `/implement-plan`) is the full user prompt describing what to implement.

Example invocations:
- `/implement-plan Add pagination to the users API endpoint`
- `/implement-plan NOH-882 — fix wrapping in order memo line`

If the prompt references a Linear issue, GitHub issue, or ticket number, fetch its full description and acceptance criteria before proceeding, and incorporate that context into the SPEC.

---

## What to do

### 1. Determine slug

Generate a short kebab-case slug from the prompt (e.g., `add-users-pagination`, `fix-order-memo-wrap`). Check whether `~/.claude/runs/implement/<slug>/` already exists. If it does, append `-2`, `-3`, etc. until the slug is unique.

### 2. Assess complexity

Decide whether the task is **simple** (3 agents) or **complex** (5 agents):

- **Simple (3):** single-file change, clear bug with obvious fix, narrow scope, no cross-cutting concerns
- **Complex (5):** multi-file changes, architectural impact, ambiguous or multi-part requirements, touches auth/payments/data integrity, migrations, or cross-cutting concerns

Print:
```
[plan] agent count: N — simple|complex (<one-line reason>)
```

### 3. Identify the target repo

Determine the git repo root the implementation will operate in. This is the current working directory (or the repo root containing it). Record the absolute path and the basename (`project`).

### 4. Scan for project Claude config

Before writing the SPEC, search the repo root for Claude configuration that should govern the implementation:

1. **Find all `CLAUDE.md` files** — search recursively (`find <repo_root> -name "CLAUDE.md" -not -path "*/node_modules/*" -not -path "*/.git/*"`). Read each one. These contain project-specific coding standards, architectural decisions, and workflow instructions that agents must follow.

2. **Check for a `.claude/` directory** at the repo root:
   - `.claude/CLAUDE.md` or `.claude/settings.json` — additional config
   - `.claude/skills/` — project-specific skills agents should be aware of (list them by name; do not copy them, just note they exist and what they do based on their SKILL.md descriptions)

3. **Synthesize the findings** into a `## Project Guidelines` section to be included in the SPEC. This section should:
   - Summarize the key rules and constraints from all `CLAUDE.md` files found (attribute each to its file path)
   - List any project skills available (e.g. "Project has a `/deploy` skill — do not manually deploy")
   - Call out any workflow requirements (e.g. testing conventions, commit message format, forbidden patterns)
   - If nothing was found, omit this section entirely

Print:
```
[plan] ✓ claude config — found N CLAUDE.md file(s), M project skill(s)
```
or:
```
[plan] ✓ claude config — none found
```

### 5. Create the run directory  <!-- renumbered: was 4, config scan inserted above -->

Create `~/.claude/runs/implement/<slug>/` and write two files:

**`README.md`** — human-readable summary of goals, background, and requirements. Written for a future reader who wants to understand *why* this work was done. Include any ticket context fetched.

**`SPEC.md`** — the full optimized prompt that will drive the implementation agents. This should:
- Clarify any ambiguities in the original prompt
- Break the work into concrete, ordered steps
- Call out risks, constraints, and edge cases
- Specify affected files/systems where known
- Include ticket acceptance criteria if applicable
- Include the `## Project Guidelines` section synthesized in step 4 (if any config was found)

### 6. Write STATE.json

Write `~/.claude/runs/implement/<slug>/STATE.json`:
```json
{
  "slug": "<slug>",
  "prompt": "<original prompt text>",
  "agent_count": 3,
  "project": "<repo-basename>",
  "repo_root": "<absolute-path-to-repo-root>",
  "step": "plan_done",
  "review_iteration": 0,
  "winning_agent": null,
  "branch": null,
  "pr_url": null
}
```

### 6. Write the current pointer

Write the slug to `~/.claude/runs/implement/.current` (overwriting any previous value).

### 7. Output

Print to stdout (required — autofix workflow parses this):
```
IMPL_SLUG: <slug>
```

Then print a human-readable summary:
```
[plan] ✓ run dir → /Users/<user>/.claude/runs/implement/<slug>
[plan] ✓ spec    → /Users/<user>/.claude/runs/implement/<slug>/SPEC.md
[plan] ✓ agents  → N (simple|complex)
[plan] ✓ project → <project> at <repo_root>
[plan] ✓ STATE   → plan_done
```

### 8. Write GitHub step summary

If `$GITHUB_STEP_SUMMARY` is set (i.e., running in GitHub Actions), append to it:

```markdown
## Plan

**Slug:** `<slug>`
**Agents:** N (simple|complex — reason)
**Project:** `<project>`

### SPEC

<contents of SPEC.md>
```

---

## Notes
- This stage does NOT touch any source code — it only creates run artifacts
- The SPEC.md is the authoritative prompt for all subsequent agents; invest time in making it precise
- If the original prompt is ambiguous, clarify in the SPEC rather than asking the user
