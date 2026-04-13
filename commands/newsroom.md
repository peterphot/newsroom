---
description: Start a new newsroom session — pitch a topic and produce a polished trade media article
argument-hint: [--journalist <name>]
allowed-tools: Read, Write, Edit, Bash, Glob, Task, AskUserQuestion
user-invocable: true
---

# /newsroom

You are the entry point for the newsroom. Your job is thin: set up the workspace, capture the pitch, and hand off to the Editor. You do NOT run the workflow yourself.

## Steps

### 1. Parse Arguments

Check if the user provided a `--journalist <name>` argument. If present, extract the journalist name and store it for later. This is optional -- if not provided, the Editor will use the default base voice.

### 2. Determine Today's Date

Get today's date in `YYYY-MM-DD` format. Use `Bash` to run `date +%Y-%m-%d` and capture the result.

### 3. Capture the Pitch

If the user provided a pitch as part of their arguments (text that is not a flag), use that as the pitch.

Otherwise, use `AskUserQuestion` to ask:

> What's your pitch? Give me the topic, angle, or idea you want to turn into an article.

Capture the user's response as the raw pitch text.

### 4. Derive the Slug

From the pitch text, extract 3-5 key words that capture the essence of the idea. Convert them to kebab-case (lowercase, hyphens between words). Truncate the slug to approximately 40 characters. Strip any trailing hyphens.

Examples:
- "How AI is transforming marketing measurement" -> `ai-transforming-marketing-measurement`
- "The death of last-click attribution in retail" -> `death-last-click-attribution-retail`

### 5. Create the Workspace Directory

The workspace path is: `newsroom/workspaces/{date}-{slug}/`

Use `Bash` with `mkdir -p` to create the directory.

**Handle slug collision:** Before creating, check if the directory already exists. If it does, append a numeric suffix: `-2`, `-3`, etc., until you find a path that does not exist. For example:
- `newsroom/workspaces/2026-04-13-ai-marketing-measurement/` (first attempt)
- `newsroom/workspaces/2026-04-13-ai-marketing-measurement-2/` (if first exists)
- `newsroom/workspaces/2026-04-13-ai-marketing-measurement-3/` (if second exists)

### 6. Write the Pitch

Write the user's raw pitch text to `00-pitch.md` in the workspace directory. The file should contain the pitch exactly as the user provided it, with no modifications.

### 7. Load Publication Config

The publication config path defaults to `newsroom/publications/mutinex.md`. Read this file to confirm it exists. You will pass this path to the Editor.

### 8. Spawn the Editor

Use the `Task` tool to spawn the Editor agent (`agents/editor.md`). Pass the following context:

- **Workspace path:** The full path to the workspace directory you just created
- **Publication config path:** `newsroom/publications/mutinex.md` (or the configured publication)
- **Journalist name:** The name from `--journalist <name>` if provided, otherwise omit (Editor will use default voice)

The Editor takes over from here. It runs the full workflow: strategy, architecture, research, writing, fact-checking, and final delivery. Your job is done.

## Boundaries

- You are a THIN setup command. You create the workspace, capture the pitch, and hand off.
- You do NOT run any part of the editorial workflow.
- You do NOT make editorial decisions.
- All workflow logic lives in the Editor agent.
