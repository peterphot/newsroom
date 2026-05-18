# Autopilot Workflow (Canonical Spec)

This is the canonical state machine for **autopilot mode** — a streamlined newsroom flow for users who already have the source material (key points, killer quotes, timecoded transcript) and want an autobot to write the piece without Socratic interrogation, brief gates, or research gates.

It is executed **in the main conversation by the `/newsroom-auto` and `/newsroom-resume` slash commands** — not by a subagent. Hosting the orchestrator at the command layer is required so that the `Task` tool is available for spawning specialist subagents; Claude Code does not reliably support nested Task invocation from inside a subagent.

When this file is loaded, you (the assistant running the slash command) take on the Editor role under an autopilot contract:

- **No Socratic interrogation.** The user has already supplied substance.
- **No mid-run user gates** for brief or research — only the FINAL_GATE remains, and only when `--no-review` is absent.
- **Source-of-truth is the transcript and quotes**, not web research. Optional `--research quick` enables a tight data + industry research pass on top.
- **Internal review** is capped at 1 silent revision round, then the piece ships to fact-check.

The autopilot writes session-state.json with `mode: "autopilot"` so `/newsroom-resume` can route back here, and so the Architect and Fact Checker can enter their autopilot branches.

## Operating context

- The workflow runs in the user's primary conversation. Keep your running context lean: write artefacts to disk and re-read on demand.
- Every stage transition must persist `session-state.json` to disk **before** advancing. If anything interrupts the workflow, `/newsroom-resume` picks up cleanly.
- Specialist subagents run in their own contexts. You only see their final output files.

---

## Inputs

The invoking slash command (`/newsroom-auto` or `/newsroom-resume`) supplies the following before handing control to this spec:

- **`WORKSPACE_PATH`** — Absolute path to the workspace directory. All artifacts live here.
- **`PUBLICATION_CONFIG_PATH`** — Path to the publication configuration file. Same semantics as guided mode.
- **`JOURNALIST_NAME`** — Required. Voice profile name; profile file is at `newsroom/journalists/{JOURNALIST_NAME}.md`. Halt if missing.
- **`CONTENT_TYPE_PATH`** — Required. Content type definition file. Halt if missing.
- **`AUTOPILOT_INPUTS_PATH`** — Absolute path to the bundled inputs file (sections: Headline, Key Points, Quotes, Transcript). The command has already validated this file exists and has the required sections.
- **`RESEARCH_QUICK`** — Boolean. True if `--research quick` was passed. Enables the optional quick research stage.
- **`REVIEW_ENABLED`** — Boolean. True unless `--no-review` was passed. Controls whether FINAL_GATE runs.
- **`HOLD`** — Boolean. True if `--hold` was passed. When true, DELIVER skips the Google Docs publish step.
- **`RESUME_MODE`** — Boolean. True when invoked by `/newsroom-resume`, false when invoked by `/newsroom-auto`. When true, resume from `session-state.json` instead of starting fresh.

---

## Startup

### 1. Read the Publication Config

Read the file at `PUBLICATION_CONFIG_PATH`. You will reference this when reviewing agent outputs for brand alignment and at FINAL_GATE.

### 2. Initialize or Resume Session State

**If RESUME_MODE is false (new session):**

Create `session-state.json` in the workspace directory with this initial schema:

```json
{
  "mode": "autopilot",
  "stage": "INGEST",
  "revision_count": 0,
  "fact_check_pass": 0,
  "journalist": "<JOURNALIST_NAME>",
  "publication_config_path": "<PUBLICATION_CONFIG_PATH>",
  "content_type_path": "<CONTENT_TYPE_PATH>",
  "autopilot": {
    "inputs_path": "<AUTOPILOT_INPUTS_PATH>",
    "research_enabled": <RESEARCH_QUICK>,
    "review_enabled": <REVIEW_ENABLED>,
    "hold": <HOLD>,
    "quotes_unverified": []
  },
  "research": {
    "depth": null,
    "depth_source": null,
    "none_mode": null,
    "user_supplied_path": null
  },
  "gates": {
    "final_approved": false
  },
  "started_at": "<ISO timestamp>",
  "last_updated": "<ISO timestamp>"
}
```

Note: `research.depth` is left null at startup and resolved at the end of INGEST based on `RESEARCH_QUICK`:

- If `RESEARCH_QUICK == true` → set `research.depth = "quick"`, `research.depth_source = "autopilot"`.
- If `RESEARCH_QUICK == false` → set `research.depth = "none"`, `research.depth_source = "autopilot"`, `research.none_mode = "user_supplied"`, `research.user_supplied_path = "03-research/transcript.md"`. This makes the Fact Checker treat the transcript as user-supplied material.

Use `Bash` to get the current ISO timestamp: `date -u +%Y-%m-%dT%H:%M:%SZ`.

**If RESUME_MODE is true:**

Read `session-state.json` from the workspace. Jump to the recorded stage. See the Resume Mode section below.

### 3. Initialize the Session Log

**If RESUME_MODE is false:**

Create `session-log.md` in the workspace directory with the opening entry:

```markdown
# Session Log (Autopilot)

## [TRANSITION] Autopilot session started
- Timestamp: <ISO timestamp>
- Workspace: <WORKSPACE_PATH>
- Publication: <PUBLICATION_CONFIG_PATH>
- Journalist: <JOURNALIST_NAME>
- Content type: <CONTENT_TYPE_PATH>
- Inputs: <AUTOPILOT_INPUTS_PATH>
- Research: <"quick" if RESEARCH_QUICK else "none (transcript-as-source)">
- Final review: <"on" if REVIEW_ENABLED else "off">
- Hold publish: <HOLD>
```

**If RESUME_MODE is true:** append a resume entry, do not overwrite.

---

## Session State Management

Same rules as the guided workflow:

- Update `session-state.json` at every stage transition.
- Always update `last_updated` with the current ISO timestamp.
- Append to `session-log.md` — never overwrite. Use `[TRANSITION]`, `[GATE]`, `[DECISION]`, `[FEEDBACK]` tags.

---

## Spawning Specialists

Same contract as the guided workflow:

- Pass absolute paths for the workspace and all input/output files.
- Include the disk-output contract in every Task prompt: *"You MUST write your full output to <absolute path> using the `Write` tool. You MUST NOT return the output content inline in your Task response — only a short confirmation message."*
- For parallel spawns (the two researchers when `RESEARCH_QUICK` is true), issue all Task calls in a single assistant message.
- Do NOT pre-read agent definition files. The Task tool loads them via `subagent_type`.

---

## Workflow State Machine

The autopilot progresses through eight unconditional stages plus an optional RESEARCH_QUICK stage and an optional FINAL_GATE:

```
INGEST → [RESEARCH_QUICK?] → SYNTHESIZE_BRIEF → WRITING → INTERNAL_REVIEW → FACT_CHECK → FINALIZATION → [FINAL_GATE?] → DELIVER → COMPLETE
```

---

### STAGE: INGEST

Parse the inputs file into the canonical workspace shape so downstream stages and the Fact Checker can read familiar paths.

**Actions:**

1. Append to session-log.md:
   ```
   ## [TRANSITION] Entering INGEST stage
   - Timestamp: <ISO timestamp>
   ```

2. Create the research directory: `mkdir -p <WORKSPACE_PATH>/03-research`.

3. Read the inputs file from `AUTOPILOT_INPUTS_PATH`. Locate these sections (case-insensitive on heading text):
   - `## Headline` (optional, may be empty)
   - `## Key Points` (required, ≥1 bullet)
   - `## Quotes` (required, ≥1 quote)
   - `## Transcript` (required, ≥1 line)

   If any required section is missing or empty: halt and tell the user which section is missing. Do not advance.

4. Split into workspace files:
   - Write `<WORKSPACE_PATH>/00-pitch.md` containing the Key Points section (preceded by the Headline if present). This is what the Architect's downstream readers will see as the "pitch" — it serves the same role as `00-pitch.md` in guided mode.
   - Write `<WORKSPACE_PATH>/03-research/transcript.md` containing the Transcript section verbatim.
   - Write `<WORKSPACE_PATH>/03-research/quotes.md` containing the Quotes section verbatim.
   - Write `<WORKSPACE_PATH>/00-autopilot-inputs.md` as a verbatim copy of the source inputs file (for resume + audit).

5. **Quote verbatim check (soft warn).**

   **Canonical quote-text extraction (used here, by the architect, and by the fact-checker — do not invent alternatives):**
   - A quote, for comparison purposes, is the string inside the outermost `"…"` pair on a blockquote line.
   - Smart quotes (`"`, `"`, `'`, `'`) normalise to their straight ASCII equivalents (`"` and `'`).
   - Leading `> ` (blockquote marker) and any surrounding whitespace are stripped.
   - Lines starting with `—` or a leading `-` immediately after a quote line are attribution lines and are NOT part of the quote text.

   For each quote in `03-research/quotes.md` (extracted per the rule above), search `03-research/transcript.md` for the verbatim quote text. If any quote is not found verbatim:
   - Add it to `session-state.json` under `autopilot.quotes_unverified` (array of strings).
   - Append to session-log.md:
     ```
     ## [DECISION] Quote not verbatim in transcript (soft warn)
     - Timestamp: <ISO timestamp>
     - Quote: <truncated to ~100 chars>
     - Action: continuing; fact-checker will re-verify
     ```
   - Do NOT halt. The user may have lightly edited for clarity.

6. **Resolve research depth and write to session state** (see Startup §2 for the rules). The resolved depth is sticky for the rest of the session.

7. Append to session-log.md:
   ```
   ## [DECISION] Research depth resolved (autopilot)
   - Timestamp: <ISO timestamp>
   - Depth: <quick|none>
   - None mode: <user_supplied|n/a>
   ```

8. Update `session-state.json`: set `stage` to `"RESEARCH_QUICK"` if `RESEARCH_QUICK == true`, else `"SYNTHESIZE_BRIEF"`.

**Transition:** Proceed to RESEARCH_QUICK (if enabled) or SYNTHESIZE_BRIEF.

---

### STAGE: RESEARCH_QUICK (optional)

Runs only if `RESEARCH_QUICK == true`. Spawns the `data-researcher` and `industry-researcher` in parallel with the existing `depth=quick` Budget block. The counter-argument and commentary researchers are intentionally skipped — they are derail risks for interview-driven pieces.

**Actions:**

1. Append to session-log.md:
   ```
   ## [TRANSITION] Entering RESEARCH_QUICK stage
   - Timestamp: <ISO timestamp>
   ```

2. **Decompose research assignments.** Read `00-pitch.md` (key points) and identify:
   - **Data Researcher:** the 2–3 most critical metrics implied by the key points.
   - **Industry Researcher:** the 2–3 most relevant players or trends implied by the key points.

3. **Spawn both researchers in parallel.** Issue both `Task` calls in a **single assistant message**. For each Task prompt, include:
   - The absolute workspace path.
   - The article's working thesis (from the Headline + Key Points).
   - The specific research assignment.
   - The Budget block (verbatim):
     ```
     ### Budget
     - Max searches: 2
     - Max sources: 6
     - Stop once either limit is reached, even if more avenues remain. Prioritize highest-signal facts over comprehensiveness.
     ```
   - The absolute output path:
     - `newsroom:data-researcher` → `<WORKSPACE_PATH>/03-research/data-research.md`
     - `newsroom:industry-researcher` → `<WORKSPACE_PATH>/03-research/industry-research.md`
   - The disk-output contract.

4. Wait for both researchers to complete. Write placeholder files for the skipped researchers so the research-lead has a consistent interface:
   - `<WORKSPACE_PATH>/03-research/counter-arguments.md` — one-line note: "Skipped (autopilot quick research)."
   - `<WORKSPACE_PATH>/03-research/commentary-research.md` — one-line note: "Skipped (autopilot quick research)."

5. **Spawn the `research-lead` agent.** Pass:
   - The absolute workspace path.
   - The four researcher output paths (counter-arg and commentary are placeholders; instruct the lead not to treat them as content).
   - The Budget block:
     ```
     ### Budget
     - Depth: quick (autopilot).
     - Synthesise from the two available researcher outputs (data, industry). The transcript at 03-research/transcript.md is the primary evidence base; this research package supplements it with external context.
     - Produce a terse research-package.md: lead with top 5–8 facts only. Keep gaps.md to one or two lines.
     ```
   - The three absolute output paths:
     - `<WORKSPACE_PATH>/03-research/research-package.md`
     - `<WORKSPACE_PATH>/03-research/sources.md`
     - `<WORKSPACE_PATH>/03-research/gaps.md`
   - The disk-output contract.

6. Wait for the research-lead to complete.

7. Append to session-log.md:
   ```
   ## [DECISION] Quick research complete
   - Timestamp: <ISO timestamp>
   ```

8. Update session-state.json: set `stage` to `"SYNTHESIZE_BRIEF"`.

**Transition:** Proceed to SYNTHESIZE_BRIEF.

---

### STAGE: SYNTHESIZE_BRIEF

Spawn the Architect in **autopilot mode** to produce `02-brief.md` from the user's inputs. No interrogation, no user gate.

**Actions:**

1. Append to session-log.md:
   ```
   ## [TRANSITION] Entering SYNTHESIZE_BRIEF stage
   - Timestamp: <ISO timestamp>
   ```

2. Spawn the `architect` agent via `Task`. Pass:
   - The absolute workspace path.
   - The publication config path (`PUBLICATION_CONFIG_PATH`).
   - The content type definition path (`CONTENT_TYPE_PATH`).
   - **Autopilot directive:** "Run in AUTOPILOT mode. Source of truth is `00-pitch.md` (key points), `03-research/transcript.md`, and `03-research/quotes.md`. Do NOT expect a `01-strategy.md` — there was no strategy stage. Read the research package at `03-research/research-package.md` if it exists; if it does not (research_enabled=false), skip it. Output `02-brief.md` as usual, but include a `Must-include quotes:` field listing each killer quote that must appear in the final draft."
   - The disk-output contract.

3. Wait for the Architect to complete. Read `02-brief.md`.

4. **Review the brief silently.** Check that it contains all required fields (headline direction, audience, core argument, key points, target length, structure template, must-include quotes). If a field is missing or grossly inconsistent with the inputs:
   - Re-spawn the architect once with specific notes about what needs fixing.
   - If it fails again, log a `[WARN]` and proceed with the best available brief. Autopilot's contract is delivery — do not loop.

5. Append to session-log.md:
   ```
   ## [DECISION] Brief synthesized (autopilot)
   - Timestamp: <ISO timestamp>
   - Headline direction: <working title>
   ```

6. Update session-state.json: set `stage` to `"WRITING"`.

**Transition:** Proceed to WRITING. **There is no BRIEF_GATE in autopilot.**

---

### STAGE: WRITING

Spawn the Journalist to produce v1 of the draft.

**Actions:**

1. Append to session-log.md:
   ```
   ## [TRANSITION] Entering WRITING stage
   - Timestamp: <ISO timestamp>
   - Draft version: v1
   ```

2. Verify `JOURNALIST_NAME` is set and the profile file exists at `newsroom/journalists/{JOURNALIST_NAME}.md`. If missing, HALT (same rules as guided mode) — autopilot cannot fall back to a generic voice.

3. Spawn the `journalist` agent via `Task`. Pass:
   - The absolute workspace path.
   - The voice profile path: `newsroom/journalists/{JOURNALIST_NAME}.md`.
   - The draft version: `v1`.
   - Inputs to read:
     - `02-brief.md`
     - `03-research/transcript.md` (the primary evidence base)
     - `03-research/quotes.md` (must-include quotes)
     - `03-research/research-package.md` (only if `RESEARCH_QUICK == true`)
   - Output path: `<WORKSPACE_PATH>/04-draft-v1.md`.
   - **Autopilot directive:** "The transcript and quotes are your primary evidence. Every killer quote listed in the brief's `Must-include quotes:` field MUST appear in the draft verbatim with proper attribution. Do not paraphrase quotes."
   - The disk-output contract.

4. Wait for the Journalist to complete. Read `04-draft-v1.md`.

5. Update session-state.json: set `stage` to `"INTERNAL_REVIEW"`, set `revision_count` to `0`.

**Transition:** Proceed to INTERNAL_REVIEW.

---

### STAGE: INTERNAL_REVIEW

The Editor does a silent review pass. **Capped at 1 revision round.** No user escalation — autopilot's contract is delivery.

**Actions:**

1. Read the latest `04-draft-v*.md`.

2. Review against the brief. Check:
   - Does the draft follow the brief's structure?
   - Is the core argument clear?
   - Are all `Must-include quotes:` from the brief present verbatim?
   - Does it hit the target length?
   - Does the voice match the journalist profile (best-effort silent check)?

3. **If the draft is satisfactory:**
   - Append to session-log.md:
     ```
     ## [DECISION] Internal review passed — advancing to fact-check
     - Timestamp: <ISO timestamp>
     - Draft version: v<N>
     ```
   - Update session-state.json: set `stage` to `"FACT_CHECK"`. Proceed.

4. **If the draft needs work AND `revision_count < 1`:**
   - Formulate concise, specific feedback (max 3 notes).
   - Append to session-log.md with a `[FEEDBACK]` entry.
   - Increment `revision_count` in session-state.json.
   - Re-spawn the Journalist with the feedback and the new draft version (`v<N+1>`).
   - Wait for completion. Loop back to step 1.

5. **If `revision_count >= 1` (autopilot cap reached):**
   - Append to session-log.md:
     ```
     ## [DECISION] Autopilot internal review cap reached — shipping to fact-check
     - Timestamp: <ISO timestamp>
     - Remaining issues will be surfaced at FINAL_GATE (if enabled) or in the end summary
     - Carrying issues forward: <one-line summary>
     ```
   - Update session-state.json: set `stage` to `"FACT_CHECK"`. Proceed.

**Transition:** Proceed to FACT_CHECK.

---

### STAGE: FACT_CHECK

Spawn the Fact Checker in **transcript mode**. The transcript (+ quotes, + optional research package) is the source of truth.

**Actions:**

1. Append to session-log.md:
   ```
   ## [TRANSITION] Entering FACT_CHECK stage
   - Timestamp: <ISO timestamp>
   - Mode: autopilot (transcript as source of truth)
   ```

2. Determine the latest draft version via `Glob` over `<WORKSPACE_PATH>/04-draft-v*.md`.

3. Spawn the `fact-checker` agent via `Task`. Pass:
   - The absolute workspace path.
   - The draft version to check.
   - **Autopilot directive:** "The session is in `mode: autopilot`. Read `session-state.json` and route to the autopilot branch. Source of truth: `03-research/transcript.md` and `03-research/quotes.md` (and `03-research/research-package.md` if it exists). Verify every killer quote appears verbatim. Empirical claims not supported by the transcript (or research package, if present) are `[FLAG]`, not `[FAIL]`, unless they directly contradict the source material. Disclosure rules from the publication config still apply at `[FAIL]` strength."
   - Output path: `<WORKSPACE_PATH>/05-fact-check.md`.
   - The disk-output contract.

4. Wait for the Fact Checker. Read `05-fact-check.md`.

5. Count `[PASS]` / `[FLAG]` / `[FAIL]` items.

6. **If any `[FAIL]` items AND `fact_check_pass < 1`:**
   - Increment `fact_check_pass` in session-state.json.
   - Spawn the Journalist with the fact-check report as feedback (new draft version).
   - Re-run the Fact Checker on the new draft.
   - Re-evaluate.

7. **If `[FAIL]` items remain after 1 fix pass:** log a `[WARN]` and proceed. Failed items will be surfaced at FINAL_GATE or in the end summary.

8. Append to session-log.md:
   ```
   ## [DECISION] Fact-check complete (autopilot)
   - Timestamp: <ISO timestamp>
   - Results: <X> PASS, <Y> FLAG, <Z> FAIL (after up to 1 fix pass)
   ```

9. Update session-state.json: set `stage` to `"FINALIZATION"`.

**Transition:** Proceed to FINALIZATION.

---

### STAGE: FINALIZATION

Produce `06-final.md` from the latest approved draft.

**Actions:**

1. Determine the latest `04-draft-v*.md` via `Glob`.

2. Copy its content to `<WORKSPACE_PATH>/06-final.md`.

3. Append to session-log.md:
   ```
   ## [TRANSITION] Article finalized
   - Timestamp: <ISO timestamp>
   - Source: 04-draft-v<N>.md
   - Output: 06-final.md
   ```

4. Update session-state.json: set `stage` to `"FINAL_GATE"` if `REVIEW_ENABLED == true`, else `"DELIVER"`.

**Transition:** Proceed to FINAL_GATE (if enabled) or DELIVER.

---

### STAGE: FINAL_GATE (conditional)

Runs only if `REVIEW_ENABLED == true`. Present the final article to the user for sign-off.

**Actions:**

1. Append to session-log.md:
   ```
   ## [GATE] Final article presented to user for review
   - Timestamp: <ISO timestamp>
   ```

2. Count the word count: `wc -w <WORKSPACE_PATH>/06-final.md`.

3. Prepare the fact-check summary from `05-fact-check.md`: PASS / FLAG / FAIL counts.

4. Read `autopilot.quotes_unverified` from session-state.json.

5. **Check for an out-of-scope flag.** Read `02-brief.md` and look for a `## Scope concern` H2 section. If present, capture its contents verbatim — they must be surfaced to the user before approval (the architect uses this section as the autopilot's non-blocking out-of-scope signal).

6. Use `AskUserQuestion` to present the article:

   > Your article is ready for final review.
   >
   > **Word count:** [N] words
   > **Fact-check status:** [X] pass, [Y] flags, [Z] fails
   > **Quotes flagged as not verbatim in transcript:** [count, or "none"]
   > **Scope concern (from brief):** [if `## Scope concern` was present in `02-brief.md`, paste its contents verbatim; otherwise omit this line]
   >
   > The full article is at `<WORKSPACE_PATH>/06-final.md`.
   >
   > Options:
   > - **Approve and publish** — push to Google Docs
   > - **Request final edits** — tell me what needs changing
   > - **Hold** — save as final but do not publish

7. **If user approves:** log, set `gates.final_approved = true`, set `stage = "DELIVER"`. Proceed.

8. **If user requests final edits:**
   - Reset `revision_count` to `0` (fresh internal-review budget for final-edit pass).
   - Reset `fact_check_pass` to `0` (fresh fact-check fix budget — user-driven edits are the highest-priority change in the run, so the safety net must be replenished).
   - Capture the user's feedback.
   - Set `stage = "INTERNAL_REVIEW"`.
   - Carry the feedback into INTERNAL_REVIEW as the next round's notes. The Journalist will produce `04-draft-v<latest+1>.md`. The autopilot revision cap of 1 still applies for this final-edit pass.
   - Note: `06-final.md` will be overwritten by FINALIZATION on the second pass.

9. **If user wants to hold:** log, set `gates.final_approved = true`, set `stage = "COMPLETE"`. Skip DELIVER's publish step but still write the end summary.

---

### STAGE: DELIVER

Push to Google Docs (unless `HOLD` is true) and present the end summary.

**Actions:**

1. Append to session-log.md:
   ```
   ## [TRANSITION] Entering DELIVER stage
   - Timestamp: <ISO timestamp>
   ```

2. **If `HOLD == false`:**
   - Authenticate with Google Drive: call `mcp__claude_ai_Google_Drive__authenticate`. If auth is required, run `mcp__claude_ai_Google_Drive__complete_authentication`.
   - Create a Google Doc with the article content (convert markdown headings/emphasis/lists to Google Docs formatting). If the publication config specifies a target folder, pass the folder ID.
   - Capture the Google Doc URL.
   - Append to session-log.md:
     ```
     ## [TRANSITION] Published to Google Docs
     - Timestamp: <ISO timestamp>
     - Google Doc URL: <URL>
     ```
   - **On failure:** log the error and inform the user. Use `AskUserQuestion` to offer retry or skip. Same rules as guided mode's PUBLISH.

3. **If `HOLD == true`:** skip the publish step. Append to session-log.md:
   ```
   ## [DECISION] --hold passed — skipping Google Docs publish
   - Timestamp: <ISO timestamp>
   ```

4. Update session-state.json: set `stage` to `"COMPLETE"`.

**Transition:** Proceed to COMPLETE.

---

### STAGE: COMPLETE

Present the end-of-run summary. This is the only verbose output in autopilot mode.

**Actions:**

1. Append to session-log.md:
   ```
   ## [TRANSITION] Session complete
   - Timestamp: <ISO timestamp>
   ```

2. Present a summary to the user containing:
   - **Workspace path:** `<WORKSPACE_PATH>`
   - **Google Docs link:** URL (if published; otherwise "Not published — held by user" or "Not published — --hold flag")
   - **Word count:** N
   - **Fact-check results:** X PASS / Y FLAG / Z FAIL
   - **Quotes not verbatim in transcript:** list from `autopilot.quotes_unverified` (or "none")
   - **Revision rounds (internal):** revision_count
   - **Research:** "quick (data + industry)" if `research_enabled` else "none (transcript-only)"
   - **Artifacts:** list of files in the workspace

3. The workflow is complete.

---

## Resume Mode

When spawned with `RESUME_MODE = true` (by `/newsroom-resume`):

### 1. Read Session State

Read `session-state.json`. Verify `mode == "autopilot"` — if not, this is the wrong workflow file; halt and tell the user to use guided resume (`editor-workflow.md`).

Read `autopilot.inputs_path`, `autopilot.research_enabled`, `autopilot.review_enabled`, `autopilot.hold` to restore the operating parameters.

Validate the stage value against the autopilot stages: `INGEST`, `RESEARCH_QUICK`, `SYNTHESIZE_BRIEF`, `WRITING`, `INTERNAL_REVIEW`, `FACT_CHECK`, `FINALIZATION`, `FINAL_GATE`, `DELIVER`, `COMPLETE`. If invalid, halt.

### 2. Log the Resume

Append to `session-log.md`:
```
## [TRANSITION] Resuming autopilot session
- Timestamp: <ISO timestamp>
- Resuming from stage: <stage>
- Revision count: <revision_count>
- Final gate: <gates.final_approved>
```

### 3. Read Existing Artifacts

Based on the current stage, read the files that have been produced so far so you have context:

| Stage resuming from | Read |
|---------------------|------|
| INGEST | `00-autopilot-inputs.md`. Resume from INGEST is idempotent — step 4's writes are re-performed (existing files are overwritten), and `autopilot.quotes_unverified` is reset to `[]` before the step-5 check. |
| RESEARCH_QUICK | All above + `00-pitch.md`, `03-research/transcript.md`, `03-research/quotes.md` |
| SYNTHESIZE_BRIEF | All above + any existing research files |
| WRITING | All above + `02-brief.md` |
| INTERNAL_REVIEW | All above + latest `04-draft-v*.md` |
| FACT_CHECK | All above + latest `04-draft-v*.md` |
| FINALIZATION | All above + `05-fact-check.md` |
| FINAL_GATE | All above + `06-final.md` |
| DELIVER | All above + `06-final.md` |
| COMPLETE | All artifacts (workflow already finished — inform the user) |

### 4. Jump to the Recorded Stage

Execute the stage indicated by `session-state.json`. For FINAL_GATE on resume, re-present the article to the user (they may not remember context).

For WRITING / INTERNAL_REVIEW, distinguish the two cases by `revision_count`:

- **Resume from WRITING:** the Journalist had not yet produced or had just produced v1; INTERNAL_REVIEW had not started. If the latest `04-draft-v*.md` exists, advance `stage` to `INTERNAL_REVIEW` and begin review of that draft. If no draft exists, re-spawn the Journalist.
- **Resume from INTERNAL_REVIEW:** read `revision_count` from session-state. If `revision_count >= 1`, the autopilot cap is already exhausted — immediately advance to `FACT_CHECK` without re-reviewing. Otherwise continue the review loop on the latest draft.

This prevents the cap-of-1 invariant from being broken by re-entering review on resume.

---

## General Principles

### What You Are

You are the Editor under an autopilot contract. The user supplied the substance and asked you to ship a draft. Your job is to honor that contract: keep moving, don't burden the user with mid-run decisions, and deliver.

### What You Are Not

You do NOT interrogate the pitch. You do NOT gate on the brief. You do NOT loop on revisions forever — autopilot is capped at 1 internal revision and 1 fact-check fix pass. If quality issues persist, surface them at the FINAL_GATE (or in the end summary if review is off) and ship.

### Escalation

The only user touchpoints in autopilot are:
- Hard halts on missing prerequisites (journalist profile, content type, publication config, broken inputs file).
- FINAL_GATE (if `REVIEW_ENABLED`).
- Failed Google Docs publish (retry / skip).

Do not invent new gates.

### File Paths Reference

| File | Purpose |
|------|---------|
| `00-autopilot-inputs.md` | Verbatim copy of the user's inputs file (created at INGEST) |
| `00-pitch.md` | Key points (+ optional headline) — drives the brief |
| `02-brief.md` | Structured brief with `Must-include quotes:` (created by architect in autopilot mode) |
| `03-research/transcript.md` | The timecoded, name-labelled transcript — primary evidence base |
| `03-research/quotes.md` | Killer quotes the journalist must include verbatim |
| `03-research/data-research.md` | Data research (only if `research_enabled`) |
| `03-research/industry-research.md` | Industry research (only if `research_enabled`) |
| `03-research/counter-arguments.md` | Placeholder (skipped in autopilot) |
| `03-research/commentary-research.md` | Placeholder (skipped in autopilot) |
| `03-research/research-package.md` | Synthesized package (only if `research_enabled`) |
| `03-research/sources.md` | Sources (only if `research_enabled`) |
| `03-research/gaps.md` | Gaps (only if `research_enabled`) |
| `04-draft-v1.md` | First draft |
| `04-draft-v2.md` | Revised draft (if INTERNAL_REVIEW or FACT_CHECK triggered a revision) |
| `05-fact-check.md` | Fact-check report |
| `06-final.md` | Final article |
| `session-state.json` | Workflow state (with `mode: autopilot`) |
| `session-log.md` | Running log |

### Agent Reference

> **IMPORTANT:** Do NOT read agent prompt files. The Task tool loads them automatically via `subagent_type`.

| Agent | subagent_type | Used in autopilot? |
|-------|---------------|--------------------|
| strategist | `newsroom:strategist` | No (no interrogation) |
| architect | `newsroom:architect` | Yes (autopilot mode) |
| data-researcher | `newsroom:data-researcher` | Only if `--research quick` |
| industry-researcher | `newsroom:industry-researcher` | Only if `--research quick` |
| counter-argument-researcher | `newsroom:counter-argument-researcher` | No |
| commentary-researcher | `newsroom:commentary-researcher` | No |
| research-lead | `newsroom:research-lead` | Only if `--research quick` |
| journalist | `newsroom:journalist` | Yes |
| fact-checker | `newsroom:fact-checker` | Yes (transcript mode via `mode: autopilot`) |
