---
description: Resume an interrupted newsroom session from where it left off
argument-hint: [workspace-path] [--depth <ignored-on-resume>]
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
- The `mode` field (`"autopilot"` or absent/`"guided"`). This determines which workflow spec to load (see Step 4).
- The current stage (e.g., PITCH, STRATEGY, ARCHITECTURE, RESEARCH, WRITING, FACT_CHECK, FINALIZATION for guided; INGEST, RESEARCH_QUICK, SYNTHESIZE_BRIEF, WRITING, INTERNAL_REVIEW, FACT_CHECK, FINALIZATION, FINAL_GATE, DELIVER for autopilot).
- Any other relevant state (revision count, journalist profile, publication config path, content type path, research depth, autopilot block, etc.)

If the user passed a `--depth` flag to `/newsroom-resume`, ignore it and warn them. Branch on `mode`:

- If `mode == "autopilot"`: warn "`--depth` does not apply in autopilot mode. Autopilot uses `--research quick` (set at session start) — research enablement is sticky across resume. Flag ignored."
- Otherwise (guided): warn "`--depth` is ignored on resume — research depth is sticky once a session starts. Continuing with depth `<research.depth from session-state.json>`."

Depth (guided) and research enablement (autopilot) can only be changed by starting a new session with `/newsroom` or `/newsroom-auto`.

If `content_type_path` is absent from session-state.json, this is a pre-content-type-selection workspace. Do not prompt the user — the Editor will fall back to `newsroom/content-types/trade-media-article.md` and log a `[WARN]` entry on its own.

## Step 3: Report to user

Tell the user which workspace is being resumed, which mode it's in, and from which stage. For example:

> Resuming newsroom session (**guided**) from workspace `newsroom/workspaces/2026-04-13-ai-marketing-trends/` at stage **RESEARCH** (depth: **standard**).

or

> Resuming newsroom session (**autopilot**) from workspace `newsroom/workspaces/2026-04-13-marketing-shift-auto/` at stage **WRITING**.

If `research.depth` is absent from `session-state.json` (pre-depth-feature workspaces), omit the depth from the message and proceed — the Editor will default to `deep` if it reaches the RESEARCH stage with no depth set.

## Step 4: Run the Workflow in resume mode

Branch on the `mode` field from `session-state.json`:

### If `mode == "autopilot"`

Read the canonical autopilot workflow spec at `${CLAUDE_PLUGIN_ROOT}/references/autopilot-workflow.md`. Execute the state machine described in that file **in this same conversation**. Use the following inputs:

- **`WORKSPACE_PATH`:** The absolute path resolved in Step 1.
- **`PUBLICATION_CONFIG_PATH`:** Read from `session-state.json` (`publication_config_path` field).
- **`JOURNALIST_NAME`:** Read from `session-state.json` (`journalist` field).
- **`CONTENT_TYPE_PATH`:** Read from `session-state.json` (`content_type_path` field).
- **`AUTOPILOT_INPUTS_PATH`:** Read from `session-state.json` (`autopilot.inputs_path` field).
- **`RESEARCH_QUICK`:** Read from `session-state.json` (`autopilot.research_enabled` field).
- **`REVIEW_ENABLED`:** Read from `session-state.json` (`autopilot.review_enabled` field).
- **`HOLD`:** Read from `session-state.json` (`autopilot.hold` field).
- **`RESUME_MODE`:** `true`.

Follow the autopilot spec's **Resume Mode** section.

### If `mode != "autopilot"` (guided, or `mode` absent on legacy workspaces)

Read the canonical Editor workflow spec at `${CLAUDE_PLUGIN_ROOT}/references/editor-workflow.md`. Execute the state machine described in that file **in this same conversation**. Use the following inputs:

- **`WORKSPACE_PATH`:** The absolute path resolved in Step 1.
- **`PUBLICATION_CONFIG_PATH`:** Read from `session-state.json` (`publication_config_path` field). The spec's Resume Mode section governs how to load it.
- **`JOURNALIST_NAME`:** Read from `session-state.json` (`journalist` field).
- **`CONTENT_TYPE_PATH`:** Read from `session-state.json` (`content_type_path` field). If absent, the spec's Resume Mode falls back to `newsroom/content-types/trade-media-article.md` and logs a `[WARN]`.
- **`DEPTH_OVERRIDE`:** `null` (ignored on resume — depth is sticky; the workflow reads `research.depth` from `session-state.json`).
- **`RESUME_MODE`:** `true`.

Follow the **Resume Mode** section of the spec. It will read `session-state.json`, validate the stage, log a resume entry, read the artefacts produced so far, and jump to the recorded stage. The spec governs everything from there.
