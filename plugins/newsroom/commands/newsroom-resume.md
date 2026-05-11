---
description: Resume an interrupted newsroom session from where it left off
argument-hint: [workspace-path]
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Task, AskUserQuestion
user-invocable: true
---

# Resume Newsroom Session

Resume an interrupted newsroom session from where it left off. Steps 1–3 locate and validate the workspace. Step 4 loads the canonical Editor workflow spec and runs it **in this conversation** — the orchestrator is hosted at the command layer (not as a subagent) so that the `Task` tool is available for spawning specialist subagents.

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
- Any other relevant state (revision count, journalist profile, publication config path, content type path, etc.)

If `content_type_path` is absent from session-state.json, this is a pre-content-type-selection workspace. Do not prompt the user — the Editor will fall back to `newsroom/content-types/trade-media-article.md` and log a `[WARN]` entry on its own.

## Step 3: Report to user

Tell the user which workspace is being resumed and from which stage. For example:

> Resuming newsroom session from workspace `newsroom/workspaces/2026-04-13-ai-marketing-trends/` at stage **RESEARCH**.

## Step 4: Run the Editor Workflow in resume mode

Read the canonical workflow spec at `${CLAUDE_PLUGIN_ROOT}/references/editor-workflow.md`. From this point on, execute the state machine described in that file **in this same conversation**. You take on the Editor role. Use the following inputs:

- **`WORKSPACE_PATH`:** The absolute path to the workspace resolved in Step 1.
- **`PUBLICATION_CONFIG_PATH`:** Read from `session-state.json` (`publication_config_path` field). The spec's Resume Mode section governs how to load it.
- **`JOURNALIST_NAME`:** Read from `session-state.json` (`journalist` field).
- **`CONTENT_TYPE_PATH`:** Read from `session-state.json` (`content_type_path` field). If absent, the spec's Resume Mode falls back to `newsroom/content-types/trade-media-article.md` and logs a `[WARN]`.
- **`RESUME_MODE`:** `true`.

Follow the **Resume Mode** section of the spec. It will read `session-state.json`, validate the stage, log a resume entry, read the artefacts produced so far, and jump to the recorded stage. The spec governs everything from there.
