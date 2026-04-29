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

For each file found, use `Read` to load it and extract a one-line descriptor.

**Descriptor extraction rule (applies to every category):** the descriptor is the first line under the named section that is not blank, not a heading (does not start with `#`), not an HTML comment (starts with `<!--`), and not a blockquote (starts with `>`).

- **`newsroom/publications/*.md`** -- look in the **Mission** section first. If it has no qualifying line (or no Mission section exists), fall back to the **Brand Voice** section. If neither yields a line, use `(no descriptor)`.
- **`newsroom/journalists/*.md`** -- look in the **Voice Summary** section. If absent, use `(no descriptor)`.
- **`newsroom/content-types/*.md`** -- look in the **Overview** section, take the first line per the rule above, then take only the text up to the first `.`, `!`, or `?` followed by whitespace or end-of-line. If no terminator is found, use the first 120 characters. If the Overview section is absent, fall back to the file's `# heading` line, otherwise `(no descriptor)`.

Truncate any descriptor to ~120 characters.

### 3. Render the Output

Print three sections, in this order. Use the file's basename (without `.md`) as the name.

If a category directory is missing or contains only `_*` files, render the section header followed by the literal line:

> `_No entries. Use /newsroom-seed-<category> to create one._`

where `<category>` is `publication`, `journalist`, or `content-type`.

Format:

```
## Publications (newsroom/publications/)

- **acme-analytics** — Acme Analytics exists to give marketing leaders evidence-based perspective on measurement...
- **widget-corp** — Widget Corp's editorial mission is to demystify industrial automation for plant managers.

## Journalists (newsroom/journalists/)

- **jane-smith** — Sharp, data-led voice with short sentences and provocative openings; writes like a frustrated insider.
- **mark-jones** — Conversational and analogy-rich; long paragraphs, deliberate pacing, anecdote-first.

## Content Types (newsroom/content-types/)

_No entries. Use /newsroom-seed-content-type to create one._
```

### 4. End

End with a hint:

> "Use `/newsroom-seed-publication`, `/newsroom-seed-journalist`, or `/newsroom-seed-content-type` to add new entries. Use `/newsroom-refine-*` to edit existing ones."
