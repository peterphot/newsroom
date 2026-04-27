---
description: Refine an existing publication config with new reference material, corrections, or section updates
argument-hint: <name>
allowed-tools: Read, Write, Edit, WebFetch, AskUserQuestion
user-invocable: true
---

# Refine Publication

Refine an existing publication config. The refiner reads the current config, summarises what it contains, asks the user what they want to change, runs a targeted interview, shows a diff before writing, and writes the file atomically.

## Process

### 1. Get the Publication Name

The publication name is provided as the argument. If absent, use `AskUserQuestion`:

> "Which publication config would you like to refine? Provide the name (the filename without `.md`)."

### 2. Read the Existing Config

Read `newsroom/publications/{name}.md`. If the file does not exist, tell the user:

> "No publication config found for '{name}'. Use `/newsroom-seed-publication {name}` to create one first."

Then **stop**.

### 3. Summarise Current State

Produce a concise summary of what the current config captures, section by section. For each section, show one line: either the first line of content, or `(empty / not specified)` if the section is a placeholder. The user needs to know what is already there before deciding what to change.

Example:

> Current config for `acme-analytics`:
> - **Mission:** Acme exists to give marketing leaders evidence-based perspective...
> - **Tone Rules:** 3 always, 3 never (with examples)
> - **Topical Scope:** In: 4 items / Out: 4 items
> - **Voice Examples:** 2 sentences
> - **Audience:** CMOs, VPs Marketing, Heads of Performance...
> - **Byline / Sign-off / CTA:** Configured (see file)
> - **Distribution Context:** 70% own / 25% trade / 5% syndication
> - **Disclosures:** Configured
> - **Style Rules:** Configured
> - **Google Docs target folder:** Not specified

### 4. Ask What to Refine

Use `AskUserQuestion`:

> "What do you want to refine? You can name specific sections, paste corrections, or provide new reference material. Examples: 'tighten the always/never tone rules', 'add a disclosure for AI-generated imagery', 'add 2 reference articles and update voice examples', 'change distribution mix'."

The user may provide any combination of: section-specific edits, new reference URLs, pasted excerpts, or general directional corrections. At least one must be provided.

### 5. Fetch New Reference Content

If new URLs were provided, use `WebFetch` to retrieve each. Retry once on failure. Note any that failed.

### 6. Targeted Interview

For each section the user has flagged for change, ask follow-up questions to draw out concrete content. Apply the same Socratic discipline as the seeder: push back on vague answers, ask for examples, surface contradictions with what is already in the config.

If new reference material conflicts with the existing voice examples or tone rules, flag the conflict and ask which one wins.

### 7. Show a Diff Before Writing

Before writing, show the user a section-by-section diff: what is being added, removed, or rewritten. Format:

```
## Tone Rules — Always
- (kept) Lead with the reader's problem. ...
- (kept) Attribute every claim. ...
+ (new) Treat the reader as a peer, not a student.

## Distribution Context
- (removed) Roughly 70% own channel, 25% trade placement, 5% syndication-first.
+ (new) Roughly 50% own channel, 40% trade placement, 10% syndication.
```

Use `AskUserQuestion`:

> "Apply these changes to `newsroom/publications/{name}.md`? Options: [Apply] [Revise the diff] [Cancel]"

If the user picks **Revise**, loop back to step 6. If **Cancel**, stop without writing. If **Apply**, continue.

### 8. Write Atomically

Use `Write` to replace `newsroom/publications/{name}.md` in a single operation with the full updated file content. Do not use multiple `Edit` calls -- one atomic write so the file is never half-updated. Preserve the section order from the seeder template; do not rearrange.

### 9. Confirm

> "Publication config refined at `newsroom/publications/{name}.md`. Refine again anytime with `/newsroom-refine-publication {name}`."
