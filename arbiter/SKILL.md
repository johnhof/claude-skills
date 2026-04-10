# Arbiter Interactive Review Documentation

The Arbiter tool enables local code diff reviews through an interactive web interface. Here's what it does:

**Core Purpose**: "Review local diff for agent guidance" when users request review of their uncommitted changes—distinct from GitHub PR reviews.

**Key Workflow Steps**:

1. **Identify Repository Details**: Determines repo path, source branch, and target branch, with sensible defaults (current directory, current branch, and `arbiter/main` respectively).

2. **Prepare Arbiter Target Branch**: Before generating the review link, hard-reset `arbiter/<primary>` to `origin/<primary>` so the comparison target is always fresh — without touching the user's local `main`:
   - Run `git fetch origin <primary> && git branch -f arbiter/<primary> origin/<primary>`
   - Then attempt `git merge arbiter/<primary> --no-edit` on the source branch to check for conflicts
   - If the merge succeeds cleanly, proceed normally.
   - If the merge results in conflicts (exit code non-zero or conflict markers present), abort the merge with `git merge --abort`, then continue to generate the URL — but prepend the following warning to your response before presenting the link:

     > **<span style="color:red">⚠ WARNING: Merge conflicts detected. This branch has conflicts with `<primary>` that must be resolved before a clean diff can be viewed. The link below may show an incomplete or misleading diff.</span>**

3. **Generate Access Link**: Creates a URL like `http://localhost:7429/?path=<repo>&source=<branch>&target=arbiter/<primary>&export=accept` that pre-loads the diff viewer.

3. **Poll for Review Comments**: Uses a bash loop to continuously check the Arbiter API until the reviewer submits comments. Poll `/api/prompts` with query params `path`, `source`, and `target` — **do NOT include `readonly=true`**, as the server uses each request to update the `lastAccess` timestamp that the UI uses to show the agent is connected. The response will be `{"error":"No prompt found"}` (404) until the user submits; once submitted it returns `{"markdown":"...","read":false}`. Loop until `"read":false` appears in the response.

4. **Process Feedback**: Reads the structured markdown response containing all comments, identifies themes, builds an implementation plan, and waits for user approval before executing changes.

5. **Handle Server Issues**: If the Arbiter server isn't running, users can start it with `arbiter &` or install via `npm install -g @johnhof/arbiter`

The interface stores comments in browser localStorage and supports multiple interaction modes, though "Accept Prompt" integration is recommended for agent workflows.
