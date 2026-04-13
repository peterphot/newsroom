# Decision Log: newsroom-claude-code-plugin

## Summary

A Claude Code plugin that runs an agentic newsroom -- 10 specialist agents orchestrated by an Editor turn a pitch into a polished trade media article delivered to Google Docs. First deployment target is Mutinex.

## User Decisions

### Research tool access
**Context**: How should researcher agents access the web?
**Choice**: Use Claude Code built-in WebSearch + WebFetch
**Why**: No extra MCP setup required. Researchers search and fetch pages directly.
**Alternatives rejected**: External search MCP tools (unnecessary dependency)

### Google Docs delivery
**Context**: How to push the final article to Google Docs?
**Choice**: Google Drive MCP (`mcp__claude_ai_Google_Drive`)
**Why**: Already available in the environment. OAuth auth, target folder configurable in publication config.

### Researcher concurrency
**Context**: Should researchers run sequentially or in parallel?
**Choice**: Parallel subagents -- Research Lead spawns all 4 simultaneously
**Why**: ~4x faster than sequential. No dependency between researchers.

### Mutinex publication config
**Context**: How complete should the Mutinex config be?
**Choice**: Scaffold with placeholders
**Why**: Template with clear sections, placeholder content. Real brand content filled separately.

### Session resume
**Context**: How should `/newsroom-resume` find the session?
**Choice**: Auto-detect latest workspace + optional path argument
**Why**: Most common use is "resume what I was just doing." Explicit path handles edge cases.

### Revision cap
**Context**: How many Editor-Journalist revision rounds before escalation?
**Choice**: 3 rounds then escalate to user
**Why**: Prevents infinite loops. User gets current draft + Editor notes.

### Voice selection
**Context**: How is a journalist voice chosen?
**Choice**: Base professional voice by default, user overrides with `--journalist <name>`
**Why**: Sensible default. Named profiles are opt-in.

## Technical Decisions

### Monolithic Editor (Approach A)
**Context**: How to structure the orchestration logic?
**Choice**: Single Editor prompt containing the entire workflow state machine (~1035 lines)
**Why**: The Editor has complete workflow context in one system prompt without requiring multi-file reads. Mirrors real newsrooms where the editor holds the full picture.
**Alternatives rejected**: Split Editor into multiple files (loses single-context advantage), distributed orchestration (too complex for a prompt-engineering project)

### 15-stage state machine
**Context**: How to model workflow progression?
**Choice**: Explicit stages: PITCH, STRATEGY, STRATEGY_GATE, ARCHITECTURE, BRIEF_GATE, RESEARCH, RESEARCH_GATE, WRITING, REVISION_LOOP, FACT_CHECK, FACT_CHECK_GATE, FINALIZATION, FINAL_GATE, PUBLISH, COMPLETE
**Why**: Explicit stages make resume trivial (read `session-state.json`, jump to the right stage). Each stage has clear entry conditions, actions, review criteria, and transitions.

### STRATEGY_GATE handled implicitly
**Context**: The Strategist already conducts user Q&A via AskUserQuestion. Should there be a separate Editor gate?
**Choice**: Editor reviews Strategist output and uses judgment to approve or send back. No separate user-facing gate.
**Why**: Adding a redundant gate after the Strategist's Socratic dialogue would double-prompt the user unnecessarily. The stage name is preserved in session-state for resume compatibility.

### RESEARCH_GATE is optional
**Context**: Should the user always review research before drafting?
**Choice**: Editor decides based on general principles ("escalate strategic decisions, handle craft internally")
**Why**: Per FR-068 and FR-102. Research quality is a craft decision the Editor can handle. Editor flags to user only if something needs their input.

### Thin command, fat Editor
**Context**: How much logic belongs in `/newsroom` vs the Editor agent?
**Choice**: Command does setup only (workspace creation, pitch capture, slug derivation). All workflow logic in the Editor.
**Why**: Single source of truth for workflow. Resume works by re-spawning the same Editor. Commands stay simple and testable.

### Files on disk for inter-agent communication
**Context**: How should agents communicate?
**Choice**: Each agent reads predecessor files and writes output files in the workspace folder
**Why**: Simple, auditable, versionable. No databases. Enables session resume from disk state.

### Structured session log
**Context**: How to maintain an audit trail?
**Choice**: Markdown file with `[GATE]`, `[FEEDBACK]`, `[DECISION]`, `[TRANSITION]` tags
**Why**: Human-readable and machine-parseable. Flexible content within structured entries.

## How to Recreate

1. Create `.claude-plugin/plugin.json` with name "newsroom", version "1.0.0".
2. Create 4 slash commands in `commands/`:
   - `newsroom.md`: Thin entry point. YAML frontmatter with `argument-hint: [--journalist <name>]`. Parses args, gets date, captures pitch, derives slug (3-5 keywords, kebab-case, ~40 chars), creates workspace at `newsroom/workspaces/{date}-{slug}/`, writes `00-pitch.md`, spawns Editor via Task.
   - `newsroom-resume.md`: Auto-detects latest workspace (by modification time) or accepts explicit path. Reads `session-state.json` to find current stage. Spawns Editor with RESUME_MODE=true.
   - `newsroom-seed-journalist.md`: Takes a name. Asks user for reference URLs/articles and style guidance. Fetches content, extracts voice characteristics (sentence structure, vocabulary, rhetoric, formality, transitions, openings/closings). Writes profile to `newsroom/journalists/{name}.md`.
   - `newsroom-refine-journalist.md`: Takes a name. Loads existing profile. Asks for new material/corrections. Integrates and rewrites the profile.
3. Create 10 agent prompts in `agents/`:
   - `editor.md` (~1035 lines): The monolithic orchestrator. 15-stage state machine. Manages `session-state.json` and `session-log.md`. Spawns all other agents via Task. Three mandatory user gates (BRIEF_GATE, revision escalation at round 3, FINAL_GATE). One optional gate (RESEARCH_GATE). Handles fact-check failures by sending draft back to Journalist. Publishes final article to Google Docs via Google Drive MCP.
   - `strategist.md`: Socratic interrogation of the pitch. Asks hard questions (What's the argument? Who cares? Why now?). Pushes back. Outputs `01-strategy.md` with Q&A log and validated topic statement.
   - `architect.md`: Takes validated topic, references content type definition. Outputs `02-brief.md` with: headline direction, audience, core argument, key points (3-5), desired outcome, tone/register, target length, structure template, research requirements.
   - `research-lead.md`: Takes brief's research requirements. Spawns 4 researchers in parallel via Task. Reads their outputs. Synthesizes into `research-package.md`, consolidates `sources.md`, flags gaps in `gaps.md`. Resolves conflicts between findings.
   - `data-researcher.md`: Statistics, data points, benchmarks. Full attribution with dates, methodology, sample sizes. Outputs `03-research/data-research.md`.
   - `industry-researcher.md`: Key players, trends, competitive landscape, analyst perspectives. Outputs `03-research/industry-research.md`.
   - `counter-argument-researcher.md`: Opposing viewpoints, critiques, logical weaknesses. Strongest case against the thesis. Outputs `03-research/counter-arguments.md`.
   - `commentary-researcher.md`: Expert quotes, thought leader perspectives, diverse voices. Outputs `03-research/commentary-research.md`.
   - `journalist.md`: Takes brief + research package + optional voice profile. Writes drafts as `04-draft-v{N}.md`. Self-reviews before submitting. Revises based on Editor feedback.
   - `fact-checker.md`: Compares draft against research. Line-by-line pass/flag/fail verification. Outputs `05-fact-check.md`. Checks factual accuracy only, not style.
4. Create publication config at `newsroom/publications/mutinex.md` with scaffolded sections: brand voice, audience, content pillars, terminology (preferred/avoid/jargon policy), style rules, Google Docs target folder.
5. Create content type definition at `newsroom/content-types/trade-media-article.md` defining article structure (hook/lede, context, argument, evidence, counterpoint, commentary, conclusion) and conventions.
6. Create empty directories: `newsroom/journalists/`, `newsroom/workspaces/`.

## Related Documents

- Requirements: docs/requirements/newsroom-claude-code-plugin.md
- Plan: docs/plans/newsroom-claude-code-plugin.md
- Raw decision log: docs/state/decisions.md
- Per-task decisions: docs/decisions/newsroom-claude-code-plugin-T*.md
