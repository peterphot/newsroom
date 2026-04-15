---
description: Start a new newsroom session — pitch a topic and produce a polished trade media article
argument-hint: [--journalist <name>] [--publication <name>]
allowed-tools: Read, Write, Edit, Bash, Glob, Task, AskUserQuestion
user-invocable: true
---

# /newsroom

You are the entry point for the newsroom. Your job is thin: ensure the newsroom directory structure exists, set up the workspace, capture the pitch, and hand off to the Editor. You do NOT run the workflow yourself.

## Steps

### 1. Initialize Newsroom Directory Structure

Check if the `newsroom/` directory exists in the current working directory. If it does not exist, scaffold the full structure:

```bash
mkdir -p newsroom/publications newsroom/content-types newsroom/journalists newsroom/workspaces
```

Then check if default template files are present. If not, create them:

**If `newsroom/content-types/trade-media-article.md` does not exist:**
Read the bundled reference file at `${CLAUDE_PLUGIN_ROOT}/references/trade-media-article.md` and copy its contents to `newsroom/content-types/trade-media-article.md` in the working directory.

**If `newsroom/publications/` is empty (no .md files):**
Read the bundled reference file at `${CLAUDE_PLUGIN_ROOT}/references/publication-template.md` and copy its contents to `newsroom/publications/_template.md` in the working directory. Tell the user:

> No publication configs found. I've created a template at `newsroom/publications/_template.md`. Copy it, rename it for your brand, and fill in your brand voice, audience, and style rules. Then run `/newsroom` again.

Then stop — do not proceed without a publication config.

### 2. Parse Arguments

Check for these optional arguments:

- `--journalist <name>` — Journalist voice profile to use. If present, extract the name.
- `--publication <name>` — Publication config to use (without path or extension). If provided, validate that `newsroom/publications/{name}.md` exists (use `Read` or `Bash`). If the file does not exist, tell the user:

  > Publication config not found: `newsroom/publications/{name}.md`. Available publications:

  Then list the `.md` files in `newsroom/publications/` (excluding `_template.md`). Stop — do not proceed.

  If not provided, auto-detect:
  - List `.md` files in `newsroom/publications/` (excluding `_template.md`)
  - If exactly one exists, use it
  - If multiple exist, use `AskUserQuestion` to ask the user which publication to use
  - If none exist (only `_template.md`), tell the user to create one from the template and stop

### 3. Determine Today's Date

Get today's date in `YYYY-MM-DD` format. Use `Bash` to run `date +%Y-%m-%d` and capture the result.

### 4. Capture the Pitch

If the user provided a pitch as part of their arguments (text that is not a flag), use that as the pitch.

Otherwise, use `AskUserQuestion` to ask:

> What's your pitch? Give me the topic, angle, or idea you want to turn into an article.

Capture the user's response as the raw pitch text.

### 5. Derive the Slug

From the pitch text, extract 3-5 key words that capture the essence of the idea. Convert them to kebab-case (lowercase, hyphens between words). Truncate the slug to approximately 40 characters. Strip any trailing hyphens.

Examples:
- "How AI is transforming marketing measurement" -> `ai-transforming-marketing-measurement`
- "The death of last-click attribution in retail" -> `death-last-click-attribution-retail`

### 6. Create the Workspace Directory

The workspace path is: `newsroom/workspaces/{date}-{slug}/`

Use `Bash` with `mkdir -p` to create the directory.

**Handle slug collision:** Before creating, check if the directory already exists. If it does, append a numeric suffix: `-2`, `-3`, etc., until you find a path that does not exist.

### 7. Write the Pitch

Write the user's raw pitch text to `00-pitch.md` in the workspace directory. The file should contain the pitch exactly as the user provided it, with no modifications.

### 8. Spawn the Editor

Use the `Task` tool to spawn the Editor agent (subagent_type: `newsroom:editor`). Pass the following context:

- **Workspace path:** The full path to the workspace directory you just created
- **Publication config path:** `newsroom/publications/{publication-name}.md`
- **Journalist name:** The name from `--journalist <name>` if provided, otherwise omit (Editor will use default voice)

The Editor takes over from here. It runs the full workflow: strategy, architecture, research, writing, fact-checking, and final delivery. Your job is done.

## Boundaries

- You are a THIN setup command. You initialize the structure, create the workspace, capture the pitch, and hand off.
- You do NOT run any part of the editorial workflow.
- You do NOT make editorial decisions.
- All workflow logic lives in the Editor agent.
