---
description: List all publications, journalists, and content types in the current newsroom
allowed-tools: Read, Bash, Glob
user-invocable: true
---

# Newsroom List

List the configurable artifacts in the current working directory's newsroom: publications, journalists, and content types. For each, show the name and a one-line descriptor pulled from the file itself.

## Process

### 1. Check the Newsroom Exists

Use `Bash` (`test -d newsroom && echo yes || echo no`) or check via `Glob` whether `newsroom/` exists. If it does not, tell the user:

> "No `newsroom/` directory in the current working directory. Run `/newsroom` first to scaffold it."

Then **stop**.

### 2. List Each Category

For each of the three category directories below, use `Glob` to find `.md` files. **Skip any file whose name starts with `_`** (these are templates, e.g., `_template.md`).

For each file found, use `Read` to load it and extract a one-line descriptor:

- **`newsroom/publications/*.md`** -- descriptor is the **Mission** section's first non-empty line. If no Mission section exists, fall back to the first non-empty line of the **Brand Voice** section. If neither exists, use `(no descriptor)`.
- **`newsroom/journalists/*.md`** -- descriptor is the first non-empty line under the **Voice Summary** section. If absent, use `(no descriptor)`.
- **`newsroom/content-types/*.md`** -- descriptor is the first sentence of the **Overview** section. If absent, use the file's `# heading` line, otherwise `(no descriptor)`.

Truncate any descriptor to ~120 characters.

### 3. Render the Output

Print three sections, in this order. Use the file's basename (without `.md`) as the name. If a category directory is missing or empty (after skipping `_*` files), say so.

Format:

```
## Publications (newsroom/publications/)

- **acme-analytics** — Acme Analytics exists to give marketing leaders evidence-based perspective on measurement...
- **widget-corp** — Widget Corp's editorial mission is to demystify industrial automation for plant managers.

## Journalists (newsroom/journalists/)

- **jane-smith** — Sharp, data-led voice with short sentences and provocative openings; writes like a frustrated insider.
- **mark-jones** — Conversational and analogy-rich; long paragraphs, deliberate pacing, anecdote-first.

## Content Types (newsroom/content-types/)

- **trade-media-article** — A long-form opinion or analysis piece written for an industry-specific audience.
- **customer-case-study** — A structured case study built around a customer outcome and the methodology that produced it.

(empty categories show as: "_No entries. Use /newsroom-seed-{category} to create one._")
```

### 4. End

End with a hint:

> "Use `/newsroom-seed-publication`, `/newsroom-seed-journalist`, or `/newsroom-seed-content-type` to add new entries. Use `/newsroom-refine-*` to edit existing ones."
