---
description: Create a new journalist voice profile from reference articles and style guidance
argument-hint: <name>
allowed-tools: Read, Write, WebFetch, AskUserQuestion
user-invocable: true
---

# Seed Journalist

Create a new journalist voice profile by analyzing reference material and style guidance.

## Process

### 1. Get the Journalist Name

The journalist name is provided as the argument to this command. If no name was provided, use `AskUserQuestion` to ask:

> "What name would you like to give this journalist voice profile?"

### 2. Gather Reference Material

Use `AskUserQuestion` to collect the following from the user:

- **Reference article URLs** -- Links to articles whose voice the user wants to replicate. Ask for at least 2-3 articles to get a representative sample.
- **Written guidance on tone, style, vocabulary, and rhetorical habits** -- Any specific direction the user has about the voice they want (e.g., "conversational but authoritative", "heavy on data", "provocative openings").
- **Example pieces or excerpts (optional)** -- The user may paste in text samples that exemplify the voice they are after.

### 3. Fetch Reference Content

Use `WebFetch` to retrieve the content of each URL the user provided. If a fetch fails, retry once. If it still fails, note the failure and continue with the URLs that succeeded.

### 4. Analyze Voice Characteristics

Read the fetched reference content and any provided excerpts. Extract the following voice characteristics:

- **Sentence structure patterns** -- Are sentences predominantly short and punchy, long and complex, or varied? Is there a rhythm?
- **Vocabulary level** -- Technical depth, jargon usage, accessibility. Does the writer assume specialist knowledge or explain terms?
- **Use of data vs anecdote** -- How does the writer use evidence? Heavy on statistics? Anecdote-driven? A mix?
- **Rhetorical devices** -- Questions to the reader, metaphors, repetition, lists, direct address, irony, understatement.
- **Formality level** -- Formal/academic, conversational, colloquial? Where on the spectrum?
- **Use of metaphor and imagery** -- Frequent or rare? What kinds of metaphors (sporting, military, domestic, technical)?
- **Paragraph length tendencies** -- Short paragraphs (1-2 sentences) or longer blocks? How does paragraph length vary?
- **How they handle transitions between ideas** -- Explicit transition phrases, abrupt shifts, logical connectives, whitespace?
- **How they open pieces** -- Hooks, cold opens, provocative statements, anecdotes, data points, questions?
- **How they close pieces** -- Calls to action, synthesis, open questions, callbacks to the opening, forward-looking statements?

### 5. Create the Journalists Directory

Create the `newsroom/journalists/` directory if it does not exist:

```bash
mkdir -p newsroom/journalists/
```

### 6. Generate the Voice Profile

Write a voice profile document to `newsroom/journalists/{name}.md` where `{name}` is the journalist name (lowercase, hyphens for spaces). The document must contain:

#### Voice Summary
2-3 sentences capturing the essence of this voice. What makes it distinctive? What would someone immediately notice?

#### Detailed Style Notes
The extracted characteristics from Step 4, organized as structured notes a writer can reference while drafting.

#### Do's and Don'ts
Specific, actionable guidance for emulating this voice:
- **Do:** concrete instructions (e.g., "Open with a bold claim, not a scene-setter")
- **Don't:** specific anti-patterns (e.g., "Don't use passive voice for attribution")

#### Example Phrases and Patterns
Characteristic expressions, sentence constructions, or turns of phrase drawn from the reference material. These are templates a writer can pattern-match against.

#### Reference Material Links
The source URLs used to create this profile, so the profile can be traced back to its origins and updated later.

## Output

When complete, confirm to the user:

> "Voice profile created at `newsroom/journalists/{name}.md`. You can use this journalist in a newsroom session with `/newsroom --journalist {name}`, or refine it later with `/newsroom-refine-journalist {name}`."
