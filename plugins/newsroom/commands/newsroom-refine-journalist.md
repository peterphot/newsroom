---
description: Refine an existing journalist voice profile with new reference material or corrections
argument-hint: <name>
allowed-tools: Read, Write, Edit, WebFetch, AskUserQuestion
user-invocable: true
---

# Refine Journalist

Refine an existing journalist voice profile by integrating new reference material or specific corrections.

## Process

### 1. Get the Journalist Name

The journalist name is provided as the argument to this command. If no name was provided, use `AskUserQuestion` to ask:

> "Which journalist voice profile would you like to refine? Provide the name."

### 2. Read the Existing Voice Profile

Read the existing voice profile from `newsroom/journalists/{name}.md` where `{name}` is the journalist name (lowercase, hyphens for spaces).

If the file does **not** exist, tell the user:

> "No voice profile found for '{name}'. Use /newsroom-seed-journalist to create one first."

Then **stop** -- do not proceed further.

### 3. Gather Refinement Input

Use `AskUserQuestion` to collect the following from the user:

- **New reference material (URLs)** -- Links to additional articles that should inform the voice profile. These help expand or adjust the stylistic model.
- **Specific corrections** -- Direct adjustments to the voice (e.g., "less jargon", "shorter sentences", "more provocative openings", "reduce use of passive voice").
- **Updated guidance** -- Any new or revised direction on tone, vocabulary, rhetorical habits, or other style dimensions.

The user may provide any combination of the above. At least one form of refinement input is required.

### 4. Fetch New Reference Content

If the user provided URLs, use `WebFetch` to retrieve the content of each URL. If a fetch fails, retry once. If it still fails, note the failure and continue with the URLs that succeeded.

### 5. Integrate Refinements with Existing Profile

Read the existing profile content carefully. Analyze the new reference material and/or corrections. Determine how they change the voice profile:

- **New reference material**: Extract additional voice characteristics, example phrases, and patterns. Merge these with existing observations -- reinforce patterns that appear in both old and new material, and note new patterns that emerge.
- **Specific corrections**: Directly adjust the relevant sections. If the user says "less jargon", update the vocabulary notes and add a "Don't" entry. If they say "shorter sentences", update sentence structure patterns.
- **Updated guidance**: Incorporate into the appropriate sections, adjusting existing notes rather than simply appending.

The voice profile is a living document that improves over time -- each refinement makes it more precise.

### 6. Rewrite the Voice Profile

Rewrite `newsroom/journalists/{name}.md`, preserving the existing structure while updating the content. The profile must retain these sections:

- **Voice Summary** -- Update to reflect any shifts from the new material or corrections.
- **Detailed Style Notes** -- Merge new observations with existing notes.
- **Do's and Don'ts** -- Add new entries from corrections; remove or revise entries that contradict new guidance.
- **Example Phrases and Patterns** -- Add new examples from new reference material; retain existing examples that remain representative.
- **Reference Material Links** -- Append any new URLs to the existing list.

Use `Write` to save the updated profile back to `newsroom/journalists/{name}.md`.

### 7. Show the User What Changed

Provide a brief summary of the updates made to the profile. For example:

> "Updated voice profile at `newsroom/journalists/{name}.md`. Changes:
> - Adjusted sentence structure notes to favor shorter sentences
> - Added 3 new example phrases from new reference articles
> - Added 'avoid technical jargon without explanation' to Don'ts
> - Appended 2 new reference links"

## Output

When complete, confirm to the user:

> "Voice profile refined at `newsroom/journalists/{name}.md`. Each refinement makes the profile more precise. You can refine again anytime with `/newsroom-refine-journalist {name}`."
