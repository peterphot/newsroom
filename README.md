# Newsroom

A Claude Code plugin that runs an agentic newsroom. Pitch a topic and a team of 10 specialist agents -- orchestrated by an Editor -- interrogates the idea, researches it, writes it, fact-checks it, and delivers a polished trade media article to Google Docs.

First deployment target: **Mutinex**.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with this repo open as a project
- Google Drive MCP configured (for final article push to Google Docs)

## Quick Start

```
/newsroom
```

Claude will ask for your pitch. The Editor takes over from there: Strategy Q&A with you, brief creation, parallel research, drafting, revision, fact-checking, and final delivery.

To use a specific journalist voice:

```
/newsroom --journalist jane-smith
```

## Slash Commands

| Command | Purpose |
|---------|---------|
| `/newsroom [--journalist <name>]` | Start a new session. Pitch a topic, produce an article. |
| `/newsroom-resume [workspace-path]` | Resume an interrupted session. Auto-detects the most recent workspace, or provide a specific path. |
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
Research Lead -- Coordinates 4 parallel researchers
  |   (Data, Industry, Counter-Argument, Commentary)
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
| **Editor** | Orchestrator. Runs the workflow state machine, gates transitions, spawns all other agents. Does not write or research. |
| **Strategist** | Interrogates the pitch with hard questions (Socratic, not helpful). Produces a validated topic statement. |
| **Architect** | Creates the structured brief: headline direction, audience, core argument, key points, tone, structure, research requirements. |
| **Research Lead** | Decomposes research needs, spawns 4 specialist researchers in parallel, synthesizes findings, flags conflicts and gaps. |
| **Data Researcher** | Statistics, data points, market figures, benchmarks. Full attribution. Objective. |
| **Industry Researcher** | Competitive landscape, trends, key players, analyst perspectives. |
| **Counter-Argument Researcher** | Opposing viewpoints, critiques, logical weaknesses. Presents the strongest case against the thesis. |
| **Commentary Researcher** | Expert quotes, thought leader perspectives, diverse voices. |
| **Journalist** | Writes drafts following the brief structure. Supports voice profiles. Revises based on Editor feedback. |
| **Fact Checker** | Compares the draft against research. Line-by-line pass/flag/fail for every substantive claim. |

## Session Workspace

Each session creates a workspace at `newsroom/workspaces/{date}-{slug}/` containing:

```
00-pitch.md              # Raw pitch
01-strategy.md           # Strategist Q&A + validated topic
02-brief.md              # Architect's structured brief
03-research/             # All research outputs
  data-research.md
  industry-research.md
  counter-arguments.md
  commentary-research.md
  research-package.md    # Synthesized findings
  sources.md             # Consolidated references
  gaps.md                # Where research was inconclusive
04-draft-v1.md           # First draft (v2, v3... for revisions)
05-fact-check.md         # Verification report
06-final.md              # Final approved article
session-state.json       # Workflow state (for resume)
session-log.md           # Full audit trail of decisions
```

Everything is Markdown. The full audit trail lives in the workspace.

## Configuration

### Publications

Publication configs live in `newsroom/publications/`. Each defines brand voice, audience, content pillars, terminology, and style rules. Mutinex is scaffolded as the first publication (`newsroom/publications/mutinex.md`).

### Content Types

Content type definitions live in `newsroom/content-types/`. These define structure and conventions for each article format. Trade media article is the v1 content type (`newsroom/content-types/trade-media-article.md`).

### Journalist Voices

Voice profiles live in `newsroom/journalists/`. Create them with `/newsroom-seed-journalist` from reference articles, then refine with `/newsroom-refine-journalist`. If no journalist is specified, the system uses a base professional voice.

## Project Structure

```
.claude-plugin/
  plugin.json                          # Plugin metadata
commands/
  newsroom.md                          # /newsroom entry point
  newsroom-resume.md                   # /newsroom-resume
  newsroom-seed-journalist.md          # /newsroom-seed-journalist
  newsroom-refine-journalist.md        # /newsroom-refine-journalist
agents/
  editor.md                            # Orchestrator (~1035 lines)
  strategist.md                        # Socratic interrogation
  architect.md                         # Brief creation
  research-lead.md                     # Research coordination
  data-researcher.md                   # Data/statistics research
  industry-researcher.md               # Industry landscape research
  counter-argument-researcher.md       # Counter-argument research
  commentary-researcher.md             # Expert commentary research
  journalist.md                        # Article drafting
  fact-checker.md                      # Fact verification
newsroom/
  publications/                        # Publication configs
    mutinex.md                         # Mutinex (scaffolded)
  content-types/                       # Content type definitions
    trade-media-article.md             # Trade media article
  journalists/                         # Voice profiles (created at runtime)
  workspaces/                          # Session workspaces (created at runtime)
```

## Architecture

This is a prompt-engineering project. Every deliverable is a Markdown file -- there is no application code. Each agent is a `.md` system prompt file that Claude Code loads as context.

The `/newsroom` command does minimal setup (creates workspace, captures pitch) then spawns the Editor agent via `Task`. The Editor contains the entire workflow as a state machine and uses `Task` to spawn other agents as subagents. All inter-agent communication happens via files on disk in the workspace folder.

## Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Editor-as-orchestrator | Monolithic state machine | Mirrors real newsrooms. Editor has judgment to gate, push back, route. Single prompt contains all workflow logic. |
| Files on disk for state | Workspace folder | Simple, auditable, versionable. No databases. Enables session resume. |
| Parallel research | 4 concurrent `Task` calls | Research Lead spawns all researchers simultaneously. ~4x faster than sequential. |
| 3-round revision cap | Escalate to user after 3 rounds | Prevents infinite loops. User gets current draft + Editor notes to decide. |
| Structured session log | Markdown with `[GATE]`, `[FEEDBACK]`, `[DECISION]`, `[TRANSITION]` tags | Machine-parseable audit trail. Flexible content within entries. |
| Graceful degradation | Continue with available data | Empty research results get flagged in `gaps.md`. Tool failures get 2 retries then note-and-continue. |
