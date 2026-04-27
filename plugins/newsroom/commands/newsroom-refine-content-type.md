---
description: Refine an existing content type definition with structural changes, convention updates, or section additions
argument-hint: <name>
allowed-tools: Read, Write, Edit, AskUserQuestion
user-invocable: true
---

# Refine Content Type

Refine an existing content type definition. Reads the current file, summarises it, runs a targeted interview, shows a diff, writes atomically.

## Process

### 1. Get the Content Type Name

The name is provided as the argument. If absent, use `AskUserQuestion`:

> "Which content type would you like to refine? Provide the name (the filename without `.md`)."

### 2. Read the Existing Definition

Read `newsroom/content-types/{name}.md`. If it does not exist:

> "No content type found for '{name}'. Use `/newsroom-seed-content-type {name}` to create one first."

Then **stop**.

### 3. Summarise Current State

Show the user a section-by-section summary of what the current file captures. For the structural template, list the section names and their proportions in order. For conventions, show one line per convention or `(empty / not specified)` if a section is a placeholder.

Example:

> Current definition for `trade-media-article`:
> - **Word count:** 800-1500
> - **Structural template:** 1. Hook/Lede (10%) → 2. Context (10-15%) → 3. Argument (5-10%) → 4. Evidence (30-35%) → 5. Counterpoint (10-15%) → 6. Commentary (10-15%) → 7. Conclusion (5-10%)
> - **Headline conventions:** Sentence case, 8-14 words, specific over clever
> - **Lede style:** Data-led, event-led, or provocation-led; no scene-setting
> - **Visual callouts:** Pull quote, data callout, sidebar
> - **Resolution style:** Single resolution -- CTA, takeaway, OR open question (no mixing)
> - **Attribution / Data / Quote / Tone:** Configured

### 4. Ask What to Refine

Use `AskUserQuestion`:

> "What do you want to change? Examples: 'add a Methodology section between Argument and Evidence', 'allow mixed resolutions', 'tighten word count to 1000-1300', 'add a chart-with-takeaway visual callout'."

At least one change is required.

### 5. Targeted Interview

For each change flagged, ask follow-ups to draw out concrete content. Apply the same Socratic discipline as the seeder: every new section needs a stated purpose distinct from its name; every new convention needs an example.

If a change creates a contradiction (e.g., adding a section that overlaps an existing one's purpose), surface the contradiction and ask which one owns the responsibility.

### 6. Show a Diff Before Writing

Show the user a section-by-section diff: what is being added, removed, or rewritten. Be explicit about structural-template changes -- if a section is being inserted, show the new ordering and adjusted proportions.

Use `AskUserQuestion`:

> "Apply these changes to `newsroom/content-types/{name}.md`? Options: [Apply] [Revise the diff] [Cancel]"

If **Revise**, loop back to step 5. If **Cancel**, stop. If **Apply**, continue.

### 7. Write Atomically

Use `Write` to replace `newsroom/content-types/{name}.md` in a single operation with the full updated file. Preserve the section order from the seeder template.

### 8. Confirm

> "Content type definition refined at `newsroom/content-types/{name}.md`. Refine again anytime with `/newsroom-refine-content-type {name}`."
