---
description: Resume an interrupted newsroom session from where it left off
argument-hint: [workspace-path]
allowed-tools: Read, Write, Bash, Glob, Task
user-invocable: true
---

# Resume Newsroom Session

Resume an interrupted newsroom session from where it left off.

## Step 1: Determine workspace path

Check if the user provided an argument (workspace path).

**If argument provided:**
- Use the provided path as the workspace path.
- Verify the path exists using `Bash` (`test -d "<path>"`). If it does not exist, tell the user: "The specified workspace path does not exist: <path>. Please check the path and try again." Then stop.

**If no argument provided:**
- Auto-detect the most recent workspace by listing directories under `newsroom/workspaces/` using `Bash` (`ls -1 newsroom/workspaces/ | sort`).
- Workspace directories use date-prefixed names (e.g., `2026-04-13-topic-slug`), so sorting by name sorts them chronologically. Take the **last** entry (most recent).
- If no directories exist under `newsroom/workspaces/`, tell the user: "No newsroom sessions found. Use /newsroom to start one." Then stop.

## Step 2: Read session state

Read `session-state.json` from the workspace directory using `Read`:

```
<workspace-path>/session-state.json
```

If `session-state.json` does not exist or cannot be read, tell the user: "Session state not found. This workspace may be corrupted." Then stop.

Parse the JSON to determine:
- The current stage (e.g., PITCH, STRATEGY, ARCHITECTURE, RESEARCH, WRITING, FACT_CHECK, FINALIZATION, etc.)
- Any other relevant state (revision count, journalist profile, etc.)

## Step 3: Report to user

Tell the user which workspace is being resumed and from which stage. For example:

> Resuming newsroom session from workspace `newsroom/workspaces/2026-04-13-ai-marketing-trends/` at stage **RESEARCH**.

## Step 4: Spawn Editor in resume mode

Use the `Task` tool to spawn the Editor agent in RESUME mode:

- **Prompt file:** `${CLAUDE_PLUGIN_ROOT}/agents/editor.md`
- **Instructions:** Pass the following context to the Editor:
  - The workspace path
  - The current stage from session-state.json
  - Instruction to resume the workflow from where the session left off (do not restart from the beginning)
  - The full contents of session-state.json so the Editor has complete state context

The Editor will pick up the workflow from the last completed stage and continue through the remaining stages.
