# Requirements: newsroom-claude-code-plugin

**Status: COMPLETED**

## Source
- **Document:** [Newsroom — Claude Code Plugin — Implementation Plan v1](https://www.notion.so/mutinex/Newsroom-Claude-Code-Plugin-Implementation-Plan-v1-341e1e39fe9181ce89eac633600ce165)
- **Type:** PRD / Implementation Plan
- **Extracted:** 2026-04-13

## Overview

A Claude Code plugin that runs an agentic newsroom. The user pitches a topic. A team of specialised agents — orchestrated by an Editor — interrogates the idea, researches it, writes it, checks it, and delivers a polished trade media article to Google Docs. The system is general-purpose (configurable per publication/brand) but the first deployment is for Mutinex.

---

## Functional Requirements

### Agents

#### Editor (Orchestrator)
- [FR-001] The Editor agent receives the initial pitch from the user.
- [FR-002] The Editor delegates to other agents at each stage of the workflow.
- [FR-003] The Editor reviews every output before it moves forward.
- [FR-004] The Editor gates each transition — can send work back, ask the user for input, or approve and advance.
- [FR-005] The Editor ensures coherence across the full piece: does the draft fulfil the brief, does the research support the claims, does the voice match the publication.
- [FR-006] The Editor maintains a session log (decisions made, rationale, feedback given).
- [FR-007] The Editor does NOT write content or research. Its job is judgment and orchestration only.
- [FR-008] The Editor has access to: publication config (brand voice, audience, content pillars, past content references), the full workspace folder for the current session, and all agent prompt files.

#### Strategist
- [FR-009] The Strategist takes the raw pitch from the user (via the Editor).
- [FR-010] The Strategist asks hard questions: What's the argument? Who cares? Why now? What's the unique angle? Has this been said before? Is this actually two ideas pretending to be one?
- [FR-011] The Strategist forces the user to sharpen their thinking — the interaction is Socratic, not helpful. It should push back.
- [FR-012] The Strategist produces a validated topic statement: a tight, clear articulation of what the piece is about and why it matters.
- [FR-013] The Strategist outputs `01-strategy.md` containing the interrogation log (Q&A with user) and the validated topic statement.
- [FR-014] The Strategist does NOT define structure or format (that is the Architect's job).

#### Architect
- [FR-015] The Architect takes the validated topic from the Strategist.
- [FR-016] The Architect determines content type and its structural requirements (for v1: trade media article).
- [FR-017] The Architect produces a structured brief that becomes the contract for all downstream work.
- [FR-018] The Architect outputs `02-brief.md` containing: headline direction (working title), audience, core argument (one sentence), key points (3-5), desired outcome, tone and register, target length (word count range), structure template (section breakdown), and research requirements.
- [FR-019] The Architect does NOT write content or do research.

#### Research Lead
- [FR-020] The Research Lead takes the brief's research requirements from the Architect.
- [FR-021] The Research Lead decomposes research into discrete tasks and assigns to specialist researchers.
- [FR-022] The Research Lead synthesises individual research outputs into a unified research package.
- [FR-023] The Research Lead identifies conflicts between researchers' findings and resolves or flags them.
- [FR-024] The Research Lead ensures the overall research is objective — guards against confirmation bias across the team.
- [FR-025] The Research Lead flags where evidence is weak or where the brief's assumptions are challenged by the data.
- [FR-026] The Research Lead outputs a `03-research/` folder containing: `research-package.md` (synthesised findings), `sources.md` (consolidated references with URLs), and `gaps.md` (areas where research was inconclusive).
- [FR-027] The Research Lead does NOT write content, editorialize, or conduct primary research. It coordinates and synthesises.

#### Data Researcher
- [FR-028] The Data Researcher searches for hard data (statistics, data points, market figures, survey results, benchmarks) that supports or challenges the brief's thesis.
- [FR-029] The Data Researcher verifies data recency and source credibility (primary sources preferred over aggregators).
- [FR-030] The Data Researcher presents data with full attribution and context (sample size, methodology where available, date of data).
- [FR-031] The Data Researcher outputs `03-research/data-research.md`.
- [FR-032] The Data Researcher does NOT interpret or spin the data — presents it objectively.

#### Industry Researcher
- [FR-033] The Industry Researcher identifies key players, companies, products, or initiatives relevant to the brief.
- [FR-034] The Industry Researcher finds recent industry moves, announcements, trends, and shifts.
- [FR-035] The Industry Researcher maps the competitive landscape: who's doing what, who's ahead, who's behind.
- [FR-036] The Industry Researcher surfaces industry analyst perspectives and commentary.
- [FR-037] The Industry Researcher identifies industry consensus on the topic and where there's disagreement.
- [FR-038] The Industry Researcher outputs `03-research/industry-research.md`.
- [FR-039] The Industry Researcher does NOT advocate for any position — maps terrain objectively.

#### Counter-Argument Researcher
- [FR-040] The Counter-Argument Researcher deliberately seeks opposing viewpoints, contradictory data, and critiques.
- [FR-041] The Counter-Argument Researcher finds published criticism, scepticism, or alternative perspectives.
- [FR-042] The Counter-Argument Researcher identifies logical weaknesses or unstated assumptions in the brief's core argument.
- [FR-043] The Counter-Argument Researcher surfaces examples where similar arguments have been made and failed or been debunked.
- [FR-044] The Counter-Argument Researcher presents the strongest possible case against the thesis — not straw men.
- [FR-045] The Counter-Argument Researcher outputs `03-research/counter-arguments.md`.
- [FR-046] The Counter-Argument Researcher does NOT conclude whether the thesis is right or wrong.

#### Commentary Researcher
- [FR-047] The Commentary Researcher identifies recognised experts, thought leaders, and practitioners with relevant perspectives.
- [FR-048] The Commentary Researcher finds published quotes, interviews, talks, or commentary from these figures.
- [FR-049] The Commentary Researcher looks for diverse perspectives — not just the usual suspects.
- [FR-050] The Commentary Researcher surfaces both supportive and critical expert commentary.
- [FR-051] The Commentary Researcher notes the credibility and relevance of each source.
- [FR-052] The Commentary Researcher outputs `03-research/commentary-research.md`.

#### Journalist
- [FR-053] The Journalist takes the brief and research package and writes a first draft following the brief's structure.
- [FR-054] The Journalist self-reviews before submitting to the Editor (proofread, check flow, verify it addresses the brief).
- [FR-055] The Journalist receives Editor feedback and revises (multiple rounds expected).
- [FR-056] The Journalist maintains voice consistency throughout — either the base voice or a seeded journalist voice.
- [FR-057] The Journalist outputs draft files in sequence: `04-draft-v1.md`, `04-draft-v2.md`, etc.
- [FR-058] Journalist agents can be created with a specific voice profile (see Journalist Persistence).

#### Fact Checker
- [FR-059] The Fact Checker compares the final draft against the research package.
- [FR-060] The Fact Checker flags: unsupported claims, stats without sources, overstatements, claims that contradict the research, hallucinated quotes or data.
- [FR-061] The Fact Checker produces a verification report: `05-fact-check.md` — line-by-line verification with pass/flag/fail for each substantive claim.
- [FR-062] The Fact Checker does NOT check grammar, style, or structure — only factual accuracy against available research.

### Workflow

- [FR-063] The workflow follows this sequence: Pitch -> Editor -> Strategist (with user Q&A) -> Editor gate -> Architect -> Editor gate (user approves brief) -> Research Lead (coordinates 4 specialist researchers) -> Editor gate -> Journalist (writes draft) -> Editor-Journalist revision loop -> Fact Checker -> Editor final review -> Google Docs push.
- [FR-064] The Editor may send work back at any gate point if quality is insufficient.

### Gate Points (User Interaction)
- [FR-065] Gate 1 — Pitch: user provides initial idea.
- [FR-066] Gate 2 — Strategy Q&A: Strategist asks questions, user answers.
- [FR-067] Gate 3 — Brief approval: user reviews and approves the Architect's brief before research starts.
- [FR-068] Gate 4 — Research review: user can see what was found (optional; Editor can flag if something needs user input).
- [FR-069] Gate 5 — Draft review: user sees the draft alongside Editor notes (optional; can let Editor-Journalist loop run).
- [FR-070] Gate 6 — Final approval: user signs off before Google Docs push.
- [FR-071] The Editor decides when to escalate to the user vs handle internally. Principle: escalate for strategic decisions (is this the right angle?), handle internally for craft decisions (this paragraph needs tightening).

### Slash Commands
- [FR-072] `/newsroom` — Main entry point. Starts a new session and creates a workspace.
- [FR-073] `/newsroom-seed-journalist` — Creates a new journalist voice profile.
- [FR-074] `/newsroom-refine-journalist` — Refines an existing journalist voice profile.
- [FR-075] `/newsroom-resume` — Resumes an interrupted session.

### Journalist Persistence

#### Seeding a New Journalist
- [FR-076] Seed-journalist command accepts inputs: name for the journalist, reference material (URLs to articles whose voice you want to replicate, and/or written guidance on tone, style, vocabulary, rhetorical habits), and optionally example pieces.
- [FR-077] System fetches and analyses reference content.
- [FR-078] System extracts voice characteristics: sentence structure patterns, vocabulary level, use of data vs anecdote, rhetorical devices, formality, use of metaphor, paragraph length tendencies, how they handle transitions, how they open and close.
- [FR-079] System generates a voice profile document saved to `newsroom/journalists/{name}.md`.
- [FR-080] Voice profile contains: voice summary (2-3 sentences), detailed style notes, do's and don'ts, example phrases/patterns, reference material links.

#### Refining a Journalist
- [FR-081] Refine-journalist command accepts: journalist name, new reference material and/or specific corrections (e.g., "less jargon", "shorter sentences", "more provocative openings").
- [FR-082] System loads existing voice profile, integrates new material/corrections, and rewrites the voice profile.
- [FR-083] The voice profile is a living document that improves over time.

### Publication Configuration
- [FR-084] Each publication gets a config file that the Editor loads at session start.
- [FR-085] Publication config contains: brand voice, audience, content pillars, terminology (preferred terms, terms to avoid, jargon policy), past content references, and style rules.
- [FR-086] A Mutinex publication config must be created as the first deployment target.

### Session Workspace
- [FR-087] Each session creates a workspace folder at `newsroom/workspaces/{date}-{slug}/`.
- [FR-088] The workspace contains all artifacts in sequence: `00-pitch.md`, `01-strategy.md`, `02-brief.md`, `03-research/` (with all sub-files), `04-draft-v*.md`, `05-fact-check.md`, `06-final.md`, and `session-log.md`.
- [FR-089] The session log is a running log of all decisions, feedback, and transitions.

### Google Docs Output
- [FR-090] The final approved article is pushed to Google Docs via MCP.
- [FR-091] Basic formatting is supported: headings, paragraphs, emphasis.

---

## Non-Functional Requirements

- [NFR-001] **Auditability:** Full paper trail of all decisions and outputs via workspace folder and session log. Git-versioned.
- [NFR-002] **Working format:** All intermediate work is in Markdown — clean, diffable, readable. Conversion to Google Docs happens at the end only.
- [NFR-003] **Simplicity:** Each agent is implemented as a `.md` system prompt file loaded as context. Simplest possible implementation — no databases needed.
- [NFR-004] **Inter-agent communication:** Agents communicate via files on disk (workspace). Simple, auditable, versionable.
- [NFR-005] **General-purpose:** The system is configurable per publication/brand via publication config files and content type definition files, even though v1 targets Mutinex.
- [NFR-006] **Content quality:** The produced article should have a clear, defensible argument; be factually grounded with real sources; read in a consistent, appropriate voice; and have required meaningful human input at the right moments (not just rubber-stamping).
- [NFR-007] **Efficiency:** The process should take less time and produce better output than writing from scratch.

---

## Technical Constraints

- [TC-001] **Platform:** Claude Code plugin (slash commands + agent prompt files).
- [TC-002] **Agent implementation:** System prompt `.md` files — each agent is a prompt file, not a separate service or binary.
- [TC-003] **Storage:** Files on disk in the repo workspace folders. No external databases.
- [TC-004] **Working format:** Markdown throughout. Google Docs conversion only at final output.
- [TC-005] **Content type for v1:** Trade media article only. Content type system is extensible but only trade media is required for v1.
- [TC-006] **First deployment target:** Mutinex. The Mutinex publication config must be the first one created.
- [TC-007] **Google Docs integration:** Via MCP (the specific MCP tool for Google Docs is not specified further).
- [TC-008] **Session resumption:** The `/newsroom-resume` command must be able to resume an interrupted session from the workspace state.

---

## Key Design Decisions (from document)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Orchestration model | Editor-as-orchestrator | Mirrors real newsrooms. Editor has judgment to gate, push back, route. |
| Inter-agent communication | Files on disk (workspace) | Simple, auditable, versionable. No databases needed. |
| User interaction model | Interactive with gates | User consulted at strategic decision points. Editor handles craft decisions autonomously. |
| Working format | Markdown | Clean, diffable, readable. Converts to Google Docs at the end only. |
| Agent implementation | System prompt files | Each agent is a .md file loaded as context. Simplest possible implementation. |
| Journalist persistence | Voice profile files | Durable, editable, version-controlled. Loaded into journalist prompt at session start. |
| Content type handling | Dedicated definition files | Each content type gets its own structure/conventions doc. Architect references these. |
| Session tracking | Workspace folder + session log | Full paper trail of all decisions and outputs. Git-versioned. |

---

## Implementation Phases (from document)

| Phase | Scope |
|-------|-------|
| Phase 1: Scaffold | Repo structure, slash command skeletons, agent prompt skeletons, publication template + Mutinex config skeleton, content type definition for trade media, workspace generation logic, verify `/newsroom` starts a session |
| Phase 2: Core Agents | Write all agent prompts: Editor, Strategist, Architect, Research Lead, 4 specialist researchers, base Journalist, Fact Checker |
| Phase 3: Orchestration | Full workflow in Editor prompt, gate points + user interaction, session logging, workspace file writes at each stage, Editor-Journalist revision loop, Research Lead delegation flow |
| Phase 4: Journalist Seeding | seed-journalist command, voice analysis logic, refine-journalist command, test with real reference content |
| Phase 5: Google Docs Output | Final push to Google Docs via MCP, basic formatting (headings, paragraphs, emphasis) |
| Phase 6: Polish + Test | End-to-end test with a real Mutinex piece, refine prompts based on output quality, stress test (bad pitches, thin research, contradictory feedback) |

---

## Edge Cases

- **Empty pitch:** User provides a vague or empty pitch — Strategist should push back with pointed questions, not proceed with nothing.
- **Researcher finds nothing:** Researcher documents the gap in their output file; Research Lead flags it in `gaps.md`. Workflow continues with available data.
- **All researchers find nothing:** Research Lead escalates to Editor, who escalates to user: "Research found insufficient material. Adjust the topic or proceed with limited evidence?"
- **User rejects brief repeatedly:** No hard cap — user iterates freely. After 3 rejections, Editor suggests reconsidering the topic angle.
- **Tool failures (WebSearch/WebFetch):** 2 retries per failure. On continued failure, researcher notes the failure and continues with available data.
- **Session interrupted mid-workflow:** `/newsroom-resume` reads workspace state and resumes from the last completed stage.
- **Conflicting research findings:** Research Lead flags conflicts in `research-package.md` and presents both sides. Journalist addresses the tension in the draft.
- **Fact checker flags critical issues:** Editor sends draft back to Journalist with fact-check report. Journalist must address all `[FAIL]` items.
- **Draft quality never converges:** After 3 Editor-Journalist revision rounds, escalate to user with current draft + Editor notes.
- **No journalist voice profile found:** Fall back to base professional voice. Log a warning in session log.
- **Workspace slug collision:** Append a numeric suffix (e.g., `-2`) if the slug already exists for that date.

## Acceptance Criteria

1. Running `/newsroom` starts a new session and creates a workspace folder at `newsroom/workspaces/{date}-{slug}/`.
2. The user can pitch a topic and the Strategist engages in Socratic Q&A to sharpen the idea.
3. The Strategist outputs `01-strategy.md` with interrogation log and validated topic statement.
4. The Architect outputs `02-brief.md` with all required fields (headline direction, audience, core argument, key points, desired outcome, tone/register, target length, structure template, research requirements).
5. The user can approve or reject the brief at the gate point.
6. The Research Lead coordinates 4 specialist researchers as parallel subagents, each producing their respective output files in `03-research/`.
7. The Research Lead produces `research-package.md`, `sources.md`, and `gaps.md`.
8. The Journalist produces drafts following the brief structure, with sequential versioning (`04-draft-v1.md`, `04-draft-v2.md`, etc.).
9. The Editor-Journalist revision loop works: Editor can send draft back with feedback and Journalist revises (capped at 3 rounds).
10. The Fact Checker produces `05-fact-check.md` with line-by-line pass/flag/fail verification against research.
11. The Editor can send the article back to the Journalist if fact-check reveals issues.
12. After final approval, the article is pushed to Google Docs via Google Drive MCP with basic formatting (headings, paragraphs, emphasis).
13. A full session log (`session-log.md`) records all decisions, feedback, and transitions using structured tags.
14. `/newsroom-resume` can resume an interrupted session from the workspace state (auto-detects latest or accepts path arg).
15. `/newsroom-seed-journalist` creates a voice profile at `newsroom/journalists/{name}.md` from reference material.
16. `/newsroom-refine-journalist` updates an existing voice profile with new material/corrections.
17. A Mutinex publication config exists (scaffolded with placeholders) and is loaded by the Editor at session start.
18. All agent prompt files exist at their specified paths under `newsroom/agents/`.
19. The final article has a clear, defensible argument, is factually grounded with real sources, reads in a consistent appropriate voice, and has a full audit trail in the workspace folder.
20. The content type definition for trade media articles exists at `newsroom/content-types/trade-media-article.md`.

---

## In Scope / Out of Scope

### In Scope
- Full agentic newsroom pipeline: pitch → strategy → brief → research → draft → fact-check → final
- 10 specialist agents (Editor, Strategist, Architect, Research Lead, 4 researchers, Journalist, Fact Checker)
- 4 slash commands: `/newsroom`, `/newsroom-seed-journalist`, `/newsroom-refine-journalist`, `/newsroom-resume`
- Mutinex as first publication target (scaffold with placeholders)
- Trade media article as the v1 content type
- Google Docs output via Google Drive MCP
- Journalist voice persistence (seed + refine)
- Session workspace with full audit trail

### Out of Scope
- Multi-journalist pieces (assigning different sections to different journalists)
- Image/visual direction (suggesting or sourcing images)
- SEO/distribution (headline optimisation, meta descriptions, distribution strategy)
- Feedback loops (performance data feeding back into journalist refinement after publication)
- Batch operations (running multiple pieces through the newsroom in parallel)
- Content calendar integration (tying into planning tools like Notion or Linear)

---

## Resolved Decisions (from clarification)

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Research tool access | Use Claude Code built-in WebSearch + WebFetch | No extra MCP setup. Researchers search and fetch pages directly. |
| 2 | Google Docs delivery | Google Drive MCP (mcp__claude_ai_Google_Drive) | Already available in environment. OAuth auth, target folder in publication config. |
| 3 | Researcher concurrency | Parallel subagents | Research Lead spawns all 4 researchers simultaneously. ~4x faster. |
| 4 | Mutinex publication config | Scaffold with placeholders | Template with clear sections, placeholder content. Real content filled separately. |
| 5 | Session resume | Auto-detect latest + optional path arg | `/newsroom-resume` auto-detects most recent workspace; user can override with explicit path. |
| 6 | Workspace slug | Auto-derived from pitch | Editor extracts 3-5 key words, kebab-case, truncated to ~40 chars. |
| 7 | Error handling | Graceful degradation | Empty results → flag in gaps.md; no cap on brief rejections; tool failures → 2 retries then note and continue. |
| 8 | Revision cap | 3 rounds then escalate | After 3 Editor-Journalist rounds, escalate to user with draft + Editor notes. |
| 9 | Voice selection | Base voice default, user overrides | Generic professional voice by default. User specifies named profile via `/newsroom --journalist <name>`. |
| 10 | Editor heuristics | General principles | "Escalate strategic decisions, handle craft internally." No rigid thresholds. |
| 11 | Session log format | Structured markdown with tags | Use `[GATE]`, `[FEEDBACK]`, `[DECISION]` tags but flexible content within entries. |

### Additional Requirements (from clarification)

- [FR-092] Specialist researchers use Claude Code built-in WebSearch for queries and WebFetch for fetching page content. No external MCP required.
- [FR-093] Google Docs push uses the Google Drive MCP (`mcp__claude_ai_Google_Drive`). Authentication via OAuth.
- [FR-094] The Research Lead spawns all 4 specialist researchers as parallel subagents and waits for all to complete before synthesizing.
- [FR-095] The Mutinex publication config is scaffolded with placeholder content. Actual brand voice, audience, pillars, and terminology are filled in separately.
- [FR-096] `/newsroom-resume` auto-detects the most recent workspace. Accepts an optional path argument to resume a specific session.
- [FR-097] The workspace slug is auto-derived by the Editor from the pitch: 3-5 key words, kebab-case, truncated to ~40 characters.
- [FR-098] When a researcher finds no relevant data, they document the gap in their output and the Research Lead flags it in `gaps.md`. The workflow continues.
- [FR-099] Tool failures (WebSearch, WebFetch) get 2 retries. On continued failure, the researcher notes the failure in their output and continues with available data.
- [FR-100] The Editor-Journalist revision loop is capped at 3 rounds. After 3 rounds, the Editor escalates to the user with the current draft and Editor notes.
- [FR-101] Base journalist voice (generic professional) is the default. User can specify a named journalist profile via `--journalist <name>` argument to `/newsroom`.
- [FR-102] The Editor prompt uses general principles for escalation decisions ("escalate strategic decisions, handle craft decisions internally") rather than rigid heuristics.
- [FR-103] The session log uses structured markdown with tags (`[GATE]`, `[FEEDBACK]`, `[DECISION]`, `[TRANSITION]`) and flexible content per entry.

### Remaining Open Questions (non-blocking)

- **Trade media article structure:** Content type definition will be derived from industry norms (hook/lede, context, argument, evidence, counterpoint, commentary, conclusion). To be refined during Phase 2.
- **Journalist voice profile template:** Will contain the fields listed in FR-079/FR-080 (voice summary, style notes, do's/don'ts, example phrases, reference links). Exact template designed during Phase 4.

