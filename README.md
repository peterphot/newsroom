# Newsroom

A Claude Code plugin that runs an agentic newsroom. Two modes:

- **Guided** (`/newsroom`) — pitch a topic and a team of 9 specialist agents interrogates the idea, researches it, writes it, fact-checks it, and delivers a polished trade media article to Google Docs.
- **Autopilot** (`/newsroom-auto`) — supply the substance up front (key points, killer quotes, timecoded transcript) and the system ships a draft end-to-end with no Socratic interrogation, no brief gate, and no research gate. For users who already did the reporting.

Install once, run in any directory. Each directory is a separate newsroom with its own publication config, journalist voices, and workspaces.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- Google Drive MCP configured (for final article push to Google Docs)

## Quick Start

### Guided mode

```
/newsroom
```

On first run, the plugin creates the `newsroom/` directory structure and a publication config template. Fill in your brand details, then run `/newsroom` again to start a session.

To use a specific publication and journalist voice:

```
/newsroom --publication acme --journalist jane-smith
```

### Autopilot mode

If you already have the source material — key points, killer quotes, a timecoded transcript — skip the interrogation:

```
/newsroom-auto --inputs ./my-interview.md
```

`my-interview.md` is a single markdown file with four sections: `## Headline` (optional), `## Key Points`, `## Quotes`, `## Transcript`. The plugin ships a template at `references/autopilot-inputs-template.md` — `/newsroom-auto` will offer to walk you through it interactively if you don't pass `--inputs`.

Autopilot flags:

| Flag | Effect |
|------|--------|
| `--inputs <path>` | Single markdown inputs file (Headline / Key Points / Quotes / Transcript) |
| `--transcript <path>` | Hybrid: supply just the transcript file; collect the rest interactively |
| `--research quick` | Run a tight data + industry research pass on top of the transcript (off by default) |
| `--no-review` | Skip the final-approval gate. Fully hands-off run. |
| `--hold` | Don't publish to Google Docs; save locally only |

The autopilot's contract is *delivery*: no Socratic Q&A, no mid-run gates (except the optional FINAL_GATE), capped at 1 internal revision and 1 fact-check fix. Quality concerns are surfaced in the end-of-run summary, not used to block the run.

### Research depth

Not every piece needs exhaustive research. Use `--depth` to control how much research runs before writing:

| Depth | Researchers spawned | Per-researcher budget | Typical sources | Use case |
|-------|---------------------|-----------------------|-----------------|----------|
| `none` | none (RESEARCH skipped) | — | 0 (or user-supplied) | Pure opinion / commentary, fastest turnaround |
| `quick` | data + industry only | 2 searches, 6 sources each | ~10 | Reactive piece, single hook |
| `standard` | all 4 | 4 searches, 12 sources each | ~40–50 | Balanced default |
| `deep` | all 4 | unbounded | ~100–150 | Investigative / flagship pieces |

```
/newsroom --depth quick
/newsroom --depth none      # then choose model-knowledge or user-supplied mode at the strategist
/newsroom --depth deep      # current default if you omit the flag
```

If you don't pass `--depth`, the Strategist will ask you during interrogation. The chosen depth is sticky for the rest of the session — `/newsroom-resume` always uses the stored depth and ignores any flag.

## Slash Commands

| Command | Purpose |
|---------|---------|
| `/newsroom [--publication <name>] [--journalist <name>] [--content-type <name>] [--depth <none\|quick\|standard\|deep>]` | Start a new guided session. Pitch a topic, produce an article. A journalist profile is required (auto-detected if exactly one exists). `--depth` controls research effort (see Research depth below). |
| `/newsroom-auto [--inputs <path>] [--transcript <path>] [--research quick] [--no-review] [--hold] [--publication <name>] [--journalist <name>] [--content-type <name>]` | Start a new autopilot session. Supply key points / killer quotes / timecoded transcript up front; the system ships a draft end-to-end with no Socratic interrogation. |
| `/newsroom-resume [workspace-path]` | Resume an interrupted session (guided or autopilot — auto-routes based on session state). Auto-detects the most recent workspace, or provide a specific path. |
| `/newsroom-list` | List all publications, journalists, and content types in the current newsroom, with one-line descriptors. |
| `/newsroom-validate` | Statically check the contract between bundled templates and agent prompts. Run after editing templates or agent inputs. |
| `/newsroom-seed-publication <name>` | Create a publication config (mission, voice, audience, tone, scope, byline, distribution, disclosures) from a Socratic interview. |
| `/newsroom-refine-publication <name>` | Refine an existing publication config with new material, corrections, or section updates. |
| `/newsroom-seed-content-type <name>` | Create a content type definition (structure, headline / lede / callout / resolution conventions) from a Socratic interview. |
| `/newsroom-refine-content-type <name>` | Refine an existing content type definition. |
| `/newsroom-seed-journalist <name>` | Create a journalist voice profile from reference articles and style guidance. |
| `/newsroom-refine-journalist <name>` | Refine an existing voice profile with new material or corrections. |

## Workflow

### Guided mode (`/newsroom`)

```
Pitch
  |
  v
Strategist -- Socratic Q&A to sharpen the idea
  |
  v
Architect -- Creates structured brief
  |
  v
[USER APPROVES BRIEF]
  |
  v
Orchestrator (Editor role) -- Spawns 4 researchers in parallel
  |   (Data, Industry, Counter-Argument, Commentary)
  v
Research Lead -- Synthesises the 4 outputs, flags conflicts and gaps
  |
  v
Journalist -- Writes draft from brief + research
  |
  v
Editor-Journalist Revision Loop (3 rounds max)
  |
  v
Fact Checker -- Line-by-line verification against research
  |
  v
[USER FINAL APPROVAL]
  |
  v
Google Docs
```

The Editor orchestrates the entire pipeline. It gates every transition -- sending work back if quality is insufficient, escalating strategic decisions to you, and handling craft decisions autonomously.

### Autopilot mode (`/newsroom-auto`)

```
Inputs file (Headline / Key Points / Quotes / Transcript)
  |
  v
INGEST -- split inputs, soft-warn on quotes not verbatim in transcript
  |
  v
[RESEARCH_QUICK?] -- only if --research quick: data + industry researchers in parallel
  |
  v
SYNTHESIZE_BRIEF -- Architect (autopilot mode), no interrogation, no user gate
  |
  v
WRITING -- Journalist drafts v1 from brief + transcript + quotes
  |
  v
INTERNAL_REVIEW -- silent Editor pass, capped at 1 revision round
  |
  v
FACT_CHECK -- Fact Checker (transcript mode), capped at 1 fix pass
  |
  v
[FINAL_GATE?] -- on by default; suppress with --no-review
  |
  v
DELIVER -- Google Docs (unless --hold) + end-of-run summary
```

The autopilot is built for *delivery*. The user already committed to the angle by writing the key points; the system honors that commitment instead of re-litigating it. Quality concerns that survive the capped review/fix passes are surfaced at the FINAL_GATE (or in the end summary if `--no-review`).

## The Agents

| Agent | Role |
|-------|------|
| **Editor (role)** | Role played by the `/newsroom` / `/newsroom-resume` slash command at the conversation layer; not a spawnable subagent. Hosts the state machine in `plugins/newsroom/references/editor-workflow.md`. Runs the workflow, gates transitions, spawns all other agents. Does not write or research. |
| **Strategist** | Interrogates the pitch with hard questions (Socratic, not helpful). Produces a validated topic statement. |
| **Architect** | Creates the structured brief: headline direction, audience, core argument, key points, tone, structure, research requirements. |
| **Research Lead** | Reads the four researcher outputs, synthesises them into a unified research package, flags conflicts and gaps. |
| **Data Researcher** | Statistics, data points, market figures, benchmarks. Full attribution. Objective. |
| **Industry Researcher** | Competitive landscape, trends, key players, analyst perspectives. |
| **Counter-Argument Researcher** | Opposing viewpoints, critiques, logical weaknesses. Presents the strongest case against the thesis. |
| **Commentary Researcher** | Expert quotes, thought leader perspectives, diverse voices. |
| **Journalist** | Writes drafts following the brief structure. Supports voice profiles. Revises based on Editor feedback. |
| **Fact Checker** | Compares the draft against research. Line-by-line pass/flag/fail for every substantive claim. |

## Running Multiple Newsrooms

Each directory where you run `/newsroom` is an independent newsroom. The plugin creates a `newsroom/` folder in your working directory with its own:

- **Publications** -- brand configs specific to that newsroom
- **Content types** -- article structure definitions
- **Journalist voices** -- voice profiles for that newsroom's writers
- **Workspaces** -- session artifacts and audit trails

Example setup for two brands:

```
~/brands/acme-content/         # cd here, run /newsroom
  newsroom/
    publications/acme.md
    journalists/sarah.md
    workspaces/...

~/brands/widget-corp/          # cd here, run /newsroom
  newsroom/
    publications/widget-corp.md
    journalists/mark.md
    workspaces/...
```

## Directory Structure

### Plugin (installed, read-only)

```
.claude-plugin/plugin.json       # Plugin metadata
commands/                        # Slash commands (incl. /newsroom-auto)
agents/                          # Agent prompts (9 files)
references/                      # Bundled templates and workflow specs
  publication-template.md        # Example publication config
  trade-media-article.md         # Default content type definition
  editor-workflow.md             # Guided workflow state machine
  autopilot-workflow.md          # Autopilot workflow state machine
  autopilot-inputs-template.md   # Autopilot inputs file template
```

### Working Directory (created at runtime)

```
newsroom/
  publications/                  # Your brand configs
    _template.md                 # Copied from plugin on first run
    your-brand.md                # Created by /newsroom-seed-publication (or hand-edited)
  content-types/                 # Article structure definitions
    trade-media-article.md       # Copied from plugin on first run
    your-content-type.md         # Created by /newsroom-seed-content-type (optional)
  journalists/                   # Voice profiles (created by /newsroom-seed-journalist)
  workspaces/                    # Session workspaces (created per /newsroom run)
    2026-04-13-topic-slug/
      00-pitch.md
      01-strategy.md
      02-brief.md
      03-research/
      04-draft-v1.md
      05-fact-check.md
      06-final.md
      session-state.json
      session-log.md
```

## Inputs Contract

Every agent in `agents/` declares which template fields it consumes via an `<inputs>` block placed immediately after its YAML frontmatter:

```
<inputs>
  <publication>mission, tone_rules, voice_examples</publication>
  <content_type>headline_conventions, structural_template</content_type>
  <journalist_profile>voice_summary, dos_and_donts</journalist_profile>
</inputs>
```

The `/newsroom-validate` command parses the bundled templates in `references/` and verifies:

- Every declared field exists in the relevant template (errors on broken contracts)
- Every template field is consumed by at least one agent, or marked `<!-- advisory-only -->` (warns on orphans)

When you add a field to a bundled template, update the `<inputs>` declaration of the consuming agent in the same change, then run `/newsroom-validate`.

Field slugs are derived from `##` and `###` headings: strip the leading `#`s, strip any leading numeric prefix like `1. ` (digits + dot + required whitespace), lowercase, drop apostrophes and quotes (`'`, `"`, smart quotes, backticks), replace runs of non-alphanumeric characters with a single underscore, trim leading/trailing underscores. Headings must be plain text (no Markdown links or code spans). Slugs are namespaced by parent tag — `publication.mission` and `content_type.mission` are distinct. A `##` declaration transitively covers all `###` children. Worked example: `## Do's and Don'ts` → `dos_and_donts`. See `/newsroom-validate`'s **Slug Derivation** section for the full algorithm.

### Upgrading from 1.2.x

Journalist profiles seeded by 1.2.x used `####` headings. 1.3.0's validator parses `##`/`###` only — pre-1.3.0 profiles will register zero fields and the journalist agent will not find the structure it expects. To migrate, either:

- Re-run `/newsroom-seed-journalist <name>` to regenerate the profile, or
- Promote each `#### ` heading in `newsroom/journalists/*.md`: top-level sections (Voice Summary, Detailed Style Notes, Do's and Don'ts, Example Phrases and Patterns, Reference Material Links) become `## `, and the Do/Don't sub-sections under "Do's and Don'ts" become `### `.

## Architecture

This is a prompt-engineering project. Every deliverable is a Markdown file -- there is no application code. Each agent is a `.md` system prompt file that Claude Code loads as context.

The `/newsroom` command does minimal setup (scaffolds directories if needed, creates workspace, captures pitch) then loads the canonical workflow spec at `plugins/newsroom/references/editor-workflow.md` and runs the editorial state machine **in the user's primary conversation**. Hosting the orchestrator at the command layer (rather than as a subagent) is required so the `Task` tool is available for spawning specialist subagents — Claude Code does not reliably support nested Task invocation from inside a subagent. The orchestrator (playing the Editor role) spawns specialist subagents one level deep: strategist, architect, the four researchers in parallel, research-lead for synthesis, journalist, and fact-checker. All inter-agent communication happens via files on disk in the workspace folder.

`/newsroom-auto` follows the same architectural pattern but loads `plugins/newsroom/references/autopilot-workflow.md` instead. It writes `mode: "autopilot"` into `session-state.json`, which (a) lets `/newsroom-resume` route back to the autopilot spec, and (b) triggers the autopilot branches inside `agents/architect.md` (skip strategy, emit a `Must-include quotes` field in the brief) and `agents/fact-checker.md` (transcript-as-source-of-truth verification). The autopilot reuses the existing journalist, data-researcher, industry-researcher, and research-lead agents as-is — only the orchestrator and the two contextual agents (architect, fact-checker) branch on mode.

## Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Editor-as-orchestrator | State machine hosted at the slash-command layer | Mirrors real newsrooms — the Editor has judgment to gate, push back, route. Running at the command layer (not as a subagent) keeps `Task` available for spawning specialists. Single spec file (`references/editor-workflow.md`) contains all workflow logic. |
| Files on disk for state | Workspace folder | Simple, auditable, versionable. No databases. Enables session resume. |
| Parallel research | 4 concurrent `Task` calls | The orchestrator spawns all four researchers simultaneously in a single message; the research-lead synthesises the outputs. ~4x faster than sequential. |
| 3-round revision cap | Escalate to user after 3 rounds | Prevents infinite loops. User gets current draft + Editor notes to decide. |
| Installable plugin | Code separate from runtime data | Plugin installs once via marketplace. Each working directory is an independent newsroom with its own config. |
| Lazy initialization | Scaffold on first `/newsroom` run | No setup command needed. Plugin creates the directory structure and copies templates automatically. |
