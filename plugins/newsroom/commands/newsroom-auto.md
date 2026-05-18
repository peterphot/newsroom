---
description: Autopilot mode — supply transcript, quotes, and key points; the newsroom writes the piece end-to-end with no Socratic interrogation
argument-hint: [--inputs <path>] [--transcript <path>] [--research quick] [--no-review] [--hold] [--publication <name>] [--journalist <name>] [--content-type <name>]
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Task, AskUserQuestion
user-invocable: true
---

# /newsroom-auto

You are the entry point for **autopilot mode**. Unlike `/newsroom`, this flow does not interrogate the user, does not gate on the brief, and does not loop on revisions. The user supplies the substance (headline-worthy angle, key points, killer quotes, timecoded transcript) and the system ships a polished draft.

Steps 1–8 set up the workspace and resolve inputs. Step 9 loads the canonical Autopilot workflow spec and runs it **in this conversation** — the orchestrator is hosted at the command layer (not as a subagent) so that the `Task` tool is available for spawning specialist subagents.

## Steps

### 1. Initialize Newsroom Directory Structure

Check if the `newsroom/` directory exists in the current working directory. If it does not, scaffold the full structure:

```bash
mkdir -p newsroom/publications newsroom/content-types newsroom/journalists newsroom/workspaces
```

Same prerequisite logic as `/newsroom`:

- **If `newsroom/content-types/trade-media-article.md` does not exist:** copy `${CLAUDE_PLUGIN_ROOT}/references/trade-media-article.md` to that path.
- **If `newsroom/publications/` is empty:** copy `${CLAUDE_PLUGIN_ROOT}/references/publication-template.md` to `newsroom/publications/_template.md` and tell the user to run `/newsroom-seed-publication <name>` first. Stop.
- **If `newsroom/journalists/` is empty:** copy `${CLAUDE_PLUGIN_ROOT}/references/journalist-template.md` to `newsroom/journalists/_template.md` and tell the user to run `/newsroom-seed-journalist <name>` first. Stop.

The autopilot has no fallback voice — a journalist profile must be selected.

### 2. Parse Arguments

The following arguments are recognized. Anything not recognized is rejected with a clear error rather than silently treated as a pitch (autopilot does not have a pitch slot — the substance lives in the inputs file).

- `--inputs <path>` — Path to a markdown inputs file (sections: Headline, Key Points, Quotes, Transcript). If provided, validate the path exists. If it does not, tell the user and stop.

- `--transcript <path>` — Hybrid mode. Path to a transcript file. The other fields (headline, key points, quotes) are collected interactively. If the path does not exist, tell the user and stop. Mutually exclusive with `--inputs` — if both are passed, reject with: "`--inputs` and `--transcript` are mutually exclusive. Use `--inputs` for a full input file or `--transcript` for hybrid mode."

- `--research quick` — Enable the optional quick research pass (data + industry researchers, tight budget). Off by default. Only the literal value `quick` is accepted; reject other values with a clear error.

- `--no-review` — Suppress the FINAL_GATE. Without this flag, autopilot pauses for final approval before publishing. With it, autopilot ships straight to Google Docs.

- `--hold` — Skip the Google Docs publish step entirely. The article is saved to `06-final.md` locally.

- `--publication <name>` — Publication config (same semantics as `/newsroom`). If provided and not found, list available publications and stop. If not provided, auto-detect: if exactly one `.md` file exists in `newsroom/publications/` (excluding `_*.md` templates), use it; if multiple, use `AskUserQuestion`; if none, tell the user to seed one and stop.

- `--journalist <name>` — Required (same semantics as `/newsroom`). Auto-detect with the same rules. Halt if none exist.

- `--content-type <name>` — Same semantics as `/newsroom`. Auto-detect with the same rules. Halt if none exist.

If a flag value is missing (e.g., `--inputs` with no path), reject with a clear error.

### 3. Determine Today's Date

Run `date +%Y-%m-%d` and capture the result.

### 4. Resolve the Inputs File

The autopilot needs a single file at `<WORKSPACE_PATH>/00-autopilot-inputs.md` with the four sections (Headline optional, Key Points / Quotes / Transcript required). Resolve this in one of three modes:

#### Mode A — `--inputs <path>` was provided

Read the file. Validate that it contains, at minimum:
- A `## Key Points` heading (case-insensitive) with at least one non-blank list item underneath. Accept any of `-`, `*`, or `1.`/`1)` as the list marker.
- A `## Quotes` heading (case-insensitive) with at least one blockquote (line beginning with `> `) underneath.
- A `## Transcript` heading (case-insensitive) with at least one non-blank line underneath.

Heading matching is case-insensitive on the heading text only — the leading `##` is required, surrounding whitespace and trailing colons are ignored. This is the same rule the INGEST parser at `references/autopilot-workflow.md` uses.

`## Headline` is optional and may be absent or empty.

If validation fails, tell the user exactly which section is missing or empty and stop. Do not advance.

If valid, mark the file as the autopilot inputs source. You will copy it into the workspace in step 7.

#### Mode B — `--transcript <path>` was provided (hybrid)

Read the transcript file. Verify it is non-empty.

Use `AskUserQuestion` to collect the other fields. Pose three questions (a single multi-question call is fine if your runtime supports it; otherwise sequential):

1. **Headline (optional).** "What's the headline or angle one-liner you want reflected? Leave blank to let the system derive one."
2. **Key Points.** "Give me 3–7 key points / themes that should drive the piece. The first should be the lead angle the headline reflects. Paste as a bulleted list or numbered list."
3. **Quotes.** "Paste the killer quotes you want included. One per blockquote with attribution on the line below, e.g., `> "Quote text" / — Name, Title, Company`. At least one quote is required."

If the user provides zero quotes or zero key points, re-prompt once. If they still fail to provide, halt with: "Autopilot requires at least one quote and at least one key point. Use `--inputs <path>` for a single-file input, or try again."

Compose the full inputs file in memory using the template at `${CLAUDE_PLUGIN_ROOT}/references/autopilot-inputs-template.md` as the structure, substituting the user's answers and the transcript content. You will write this to the workspace in step 7.

#### Mode C — No `--inputs` and no `--transcript` (fully interactive)

Use `AskUserQuestion` to ask the user how they want to provide the transcript:

> Autopilot needs a transcript. How will you provide it?
> - **File path** — I'll give you a path to a transcript file on disk (recommended for transcripts over ~500 words).
> - **Paste** — I'll paste it into chat now (fine for short transcripts).

If **File path**: prompt for the path. Verify it exists and is non-empty. Then collect Headline / Key Points / Quotes via `AskUserQuestion` as in Mode B.

If **Paste**: collect all four sections (Headline optional, Key Points, Quotes, Transcript) via `AskUserQuestion`. Validate that key points and quotes are non-empty; re-prompt once if not. Halt with the same message as Mode B if they still fail.

Either way, assemble the full inputs file in memory using the bundled template as the structure. You will write it to the workspace in step 7.

### 5. Derive the Slug

From the Headline (if present) or the first Key Point, extract 3–5 key words. Convert to kebab-case (lowercase, hyphens between words). Truncate to ~40 characters. Strip trailing hyphens.

Examples:
- Headline: "How AI is transforming marketing measurement" → `ai-transforming-marketing-measurement`
- Key Point 1: "The death of last-click attribution in retail" → `death-last-click-attribution-retail`

### 6. Create the Workspace Directory

The workspace path is: `newsroom/workspaces/{date}-{slug}-auto/`

The `-auto` suffix distinguishes autopilot workspaces from guided workspaces at a glance.

Use `Bash` with `mkdir -p` to create it. If the directory already exists, append a numeric suffix (`-2`, `-3`, …) until you find a free path.

### 7. Write the Inputs File to the Workspace

Write the resolved inputs to `<WORKSPACE_PATH>/00-autopilot-inputs.md`.

- In Mode A, copy the source file verbatim.
- In Modes B and C, write the assembled file.

This file is the canonical record of what the user supplied. The Autopilot workflow's INGEST stage parses it and splits it into `00-pitch.md`, `03-research/transcript.md`, and `03-research/quotes.md`.

### 8. Tell the User What's About to Happen

Before handing off to the workflow, print a single concise message so the user knows what to expect:

> **Autopilot session starting.**
>
> Workspace: `<WORKSPACE_PATH>`
> Publication: `<publication-name>` · Journalist: `<journalist-name>` · Content type: `<content-type-name>`
> Research: `<"quick (data + industry)" if --research quick else "none — transcript is the evidence base">`
> Final review gate: `<"on (you'll approve before publishing)" if review enabled else "off (--no-review)">`
> Publish: `<"hold (local only)" if --hold else "Google Docs">`
>
> Stages: <assemble the line below before printing — do NOT emit literal `<...>` brackets>
>   - If `--research quick` AND review enabled: `INGEST → RESEARCH_QUICK → SYNTHESIZE_BRIEF → WRITING → INTERNAL_REVIEW → FACT_CHECK → FINALIZATION → FINAL_GATE → DELIVER`
>   - If `--research quick` AND `--no-review`: same but drop `FINAL_GATE`
>   - If no research AND review enabled: drop `RESEARCH_QUICK`
>   - If no research AND `--no-review`: drop both `RESEARCH_QUICK` and `FINAL_GATE`
>
> No mid-run questions unless something is wrong.

Then proceed.

### 9. Run the Autopilot Workflow

Read the canonical workflow spec at `${CLAUDE_PLUGIN_ROOT}/references/autopilot-workflow.md`. From this point on, execute the state machine described in that file **in this same conversation**. You take on the Editor role under the autopilot contract. Use the following inputs:

- **`WORKSPACE_PATH`:** The absolute path to the workspace created in step 6.
- **`PUBLICATION_CONFIG_PATH`:** `newsroom/publications/{publication-name}.md` (resolved in step 2).
- **`JOURNALIST_NAME`:** The selected journalist (required, resolved in step 2).
- **`CONTENT_TYPE_PATH`:** `newsroom/content-types/{content-type-name}.md` (required, resolved in step 2).
- **`AUTOPILOT_INPUTS_PATH`:** `<WORKSPACE_PATH>/00-autopilot-inputs.md` (written in step 7).
- **`RESEARCH_QUICK`:** `true` if `--research quick` was passed, else `false`.
- **`REVIEW_ENABLED`:** `false` if `--no-review` was passed, else `true`.
- **`HOLD`:** `true` if `--hold` was passed, else `false`.
- **`RESUME_MODE`:** `false`.

Start at the INGEST stage. Follow the spec's stage instructions in order. The spec governs everything from here.

## Boundaries

- Steps 1–8 are pure setup — directory scaffolding, argument parsing, input resolution, workspace creation, user-facing message.
- Step 9 hands control to the workflow spec. Once loaded, that spec is the source of truth for stage logic, agent spawning, and the FINAL_GATE.
- Do NOT spawn the workflow as a subagent. Nested Task invocation is not reliably supported; the orchestrator must run at the command layer to have access to `Task`.
- Do NOT silently fall through to `/newsroom` if autopilot can't proceed. If a prerequisite is missing, halt and tell the user. They opted into autopilot deliberately.
