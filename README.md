# Newsroom

A Claude Code plugin that runs an agentic newsroom. Pitch a topic and a team of 9 specialist agents -- orchestrated by the `/newsroom` slash command playing the Editor role -- interrogates the idea, researches it, writes it, fact-checks it, and delivers a polished trade media article to Google Docs.

Install once, run in any directory. Each directory is a separate newsroom with its own publication config, journalist voices, and workspaces.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- Google Drive MCP configured (for final article push to Google Docs)

## Quick Start

```
/newsroom
```

On first run, the plugin creates the `newsroom/` directory structure and a publication config template. Fill in your brand details, then run `/newsroom` again to start a session.

To use a specific publication and journalist voice:

```
/newsroom --publication acme --journalist jane-smith
```

## Slash Commands

| Command | Purpose |
|---------|---------|
| `/newsroom [--publication <name>] [--journalist <name>]` | Start a new session. Pitch a topic, produce an article. A journalist profile is required (auto-detected if exactly one exists). |
| `/newsroom-resume [workspace-path]` | Resume an interrupted session. Auto-detects the most recent workspace, or provide a specific path. |
| `/newsroom-list` | List all publications, journalists, and content types in the current newsroom, with one-line descriptors. |
| `/newsroom-validate` | Statically check the contract between bundled templates and agent prompts. Run after editing templates or agent inputs. |
| `/newsroom-seed-publication <name>` | Create a publication config (mission, voice, audience, tone, scope, byline, distribution, disclosures) from a Socratic interview. |
| `/newsroom-refine-publication <name>` | Refine an existing publication config with new material, corrections, or section updates. |
| `/newsroom-seed-content-type <name>` | Create a content type definition (structure, headline / lede / callout / resolution conventions) from a Socratic interview. |
| `/newsroom-refine-content-type <name>` | Refine an existing content type definition. |
| `/newsroom-seed-journalist <name>` | Create a journalist voice profile from reference articles and style guidance. |
| `/newsroom-refine-journalist <name>` | Refine an existing voice profile with new material or corrections. |

## Workflow

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
commands/                        # Slash commands
agents/                          # Agent prompts (9 files)
references/                      # Bundled templates
  publication-template.md        # Example publication config
  trade-media-article.md         # Default content type definition
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

The `/newsroom` command does minimal setup (scaffolds directories if needed, creates workspace, captures pitch) then loads the canonical workflow spec at `plugins/newsroom/references/editor-workflow.md` and runs the editorial state machine **in the user's primary conversation**. Hosting the orchestrator at the command layer (rather than as a subagent) is required so the `Task` tool is available for spawning specialist subagents — Claude Code does not reliably support nested Task invocation from inside a subagent. The orchestrator (playing the Editor role) spawns specialist subagents one level deep: strategist, architect, the four researchers in parallel, research-lead for synthesis, journalist, and fact-checker. `/newsroom-resume` loads the same spec in resume mode. All inter-agent communication happens via files on disk in the workspace folder.

## Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Editor-as-orchestrator | State machine hosted at the slash-command layer | Mirrors real newsrooms — the Editor has judgment to gate, push back, route. Running at the command layer (not as a subagent) keeps `Task` available for spawning specialists. Single spec file (`references/editor-workflow.md`) contains all workflow logic. |
| Files on disk for state | Workspace folder | Simple, auditable, versionable. No databases. Enables session resume. |
| Parallel research | 4 concurrent `Task` calls | The orchestrator spawns all four researchers simultaneously in a single message; the research-lead synthesises the outputs. ~4x faster than sequential. |
| 3-round revision cap | Escalate to user after 3 rounds | Prevents infinite loops. User gets current draft + Editor notes to decide. |
| Installable plugin | Code separate from runtime data | Plugin installs once via marketplace. Each working directory is an independent newsroom with its own config. |
| Lazy initialization | Scaffold on first `/newsroom` run | No setup command needed. Plugin creates the directory structure and copies templates automatically. |
