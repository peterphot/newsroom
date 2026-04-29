---
description: Validate the static contract between bundled templates and agent prompts
allowed-tools: Read, Bash, Glob
user-invocable: true
---

# Newsroom Validate

Statically check that every agent's declared inputs exist in the bundled templates, and that every template field is consumed by at least one agent (or marked advisory-only). This is a static parse of files on disk -- it does not run the pipeline.

## The Contract

- Each bundled template in `${CLAUDE_PLUGIN_ROOT}/references/` defines a set of named sections (`##` and `###` headings).
- Each agent in `${CLAUDE_PLUGIN_ROOT}/agents/` declares which sections it consumes via an `<inputs>` block placed immediately after the YAML frontmatter:
  ```
  <inputs>
    <publication>field_a, field_b</publication>
    <content_type>field_x</content_type>
    <journalist_profile>field_p, field_q</journalist_profile>
  </inputs>
  ```
- All three child tags are required (use empty content if the agent consumes nothing from that namespace).
- A `##`-level declaration transitively covers all `###` children of that section.
- A heading is **advisory** if the very next line (no skipping blanks) is exactly the literal string `<!-- advisory-only -->`. A blank line between heading and marker means the heading is NOT advisory. Advisory headings are exempt from orphan checking.
- HTML comments other than the exact string `<!-- advisory-only -->` (e.g., `<!-- TODO -->`) are NOT advisory markers. If such a comment sits on the line directly after a heading, the heading is **not advisory** (because the strict "very next line, no skipping" rule applies — the comment is not the marker, so the marker is not present). Do NOT skip past non-advisory comments looking for a later marker; that would re-introduce the rule contradiction.
- Slugs are **namespaced by parent tag**. `publication.mission` and `content_type.mission` are distinct and do not collide. Cross-namespace lookups never happen.

## Slug Derivation

For a heading `### Some Heading: Foo's & Bar`:
1. Strip the leading `#`s and whitespace.
2. Strip a leading numeric prefix like `1.` or `2.` (digits + dot + optional whitespace; the whitespace is required, so `1.Hook` would NOT have its prefix stripped — author convention is `1. Hook`).
3. Lowercase.
4. Strip these specific quote characters: `'`, `"`, `'` (U+2019), `'` (U+2018), `"` (U+201D), `"` (U+201C), `` ` `` (backtick). Smart quotes auto-inserted by editors are normalised before rule 5.
5. Replace any run of non-alphanumeric characters (other than `_`) with a single underscore.
6. Trim leading/trailing underscores.

Headings must be **plain text**: no Markdown link syntax `[text](url)`, no code spans `` `code` ``. Such headings produce undefined slugs — the validator emits a WARNING.

Examples:
- `## Mission` → `mission`
- `## Tone Rules` → `tone_rules`
- `### 1. Hook/Lede` → `hook_lede`
- `## Byline, Sign-off, and CTA Conventions` → `byline_sign_off_and_cta_conventions`
- `### Do's and Don'ts` → `dos_and_donts`
- `### "Smart" Quotes` → `smart_quotes`

## Process

### 1. Resolve Plugin Paths

Use `Bash` (`echo "${CLAUDE_PLUGIN_ROOT}"`) to get the plugin root.

If the value is unset or empty, abort immediately with:

> `ERROR: CLAUDE_PLUGIN_ROOT is not set. This command must run inside a loaded Claude Code plugin context.`

Do not proceed to template parsing.

The three template files are:

- `${CLAUDE_PLUGIN_ROOT}/references/publication-template.md` → namespace `publication`
- `${CLAUDE_PLUGIN_ROOT}/references/trade-media-article.md` → namespace `content_type`
- `${CLAUDE_PLUGIN_ROOT}/references/journalist-template.md` → namespace `journalist_profile`

If any of these files does not exist, emit an ERROR for the missing template and continue with the others (a missing journalist template, for example, means agents declaring fields in `<journalist_profile>` cannot be validated -- treat all such declarations as errors).

### 2. Parse Each Template

For each template:

1. Read the file with `Read`.
2. Walk the lines. For each line matching `^## ` or `^### `:
   - Compute the slug per the rules above.
   - Determine level (`##` = 2, `###` = 3).
   - For `###`, the parent is the most recent preceding `##`.
   - Look at the very next line — do NOT skip blanks. If that line is exactly `<!-- advisory-only -->`, mark this heading as advisory. A blank line between heading and marker means the heading is NOT advisory.
   - Add to the template's field list: `(slug, level, parent_slug_or_null, advisory)`.
3. Verify slug uniqueness within the template. If duplicates exist, emit an ERROR (`duplicate slug "X" in publication-template.md`).

### 3. Parse Each Agent

`Glob` `${CLAUDE_PLUGIN_ROOT}/agents/*.md`. For each:

1. Read the file.
2. Find the first `<inputs>...</inputs>` block (single regex: matches across lines, non-greedy).
   - If the block is missing → ERROR (`agent <name> has no <inputs> declaration`).
3. Within the block, extract the contents of `<publication>`, `<content_type>`, and `<journalist_profile>`.
   - If any of the three child tags is absent → ERROR (`agent <name> missing <child_tag> declaration -- use an empty tag if the agent consumes nothing`).
4. For each child tag's contents: split on commas, trim whitespace, drop empty entries. Result is the declared slug list.

### 4. Cross-Check (Errors)

For each agent, for each declared slug in each namespace:

- If the slug is not in the corresponding template's field list → ERROR (`agent <name> declares <namespace>.<slug>, which does not exist in <template_filename>`).

### 5. Coverage and Orphans (Warnings)

For each template field that is **not** advisory:

- Compute the set of agents that cover it. An agent covers a field if it declares the field's exact slug, OR (when the field is at level 3) if it declares the parent slug at level 2.
- If no agent covers the field → WARNING (`<namespace>.<slug> in <template_filename> is not consumed by any agent (orphan)`).

### 6. Render the Report

Print the report in this format. Use Markdown so it reads cleanly in the CLI.

```
# Newsroom Validate

## Templates

- publication-template.md: <N> fields (<advisory_count> advisory)
- trade-media-article.md: <N> fields (<advisory_count> advisory)
- journalist-template.md: <N> fields (<advisory_count> advisory)

## Agents

<for each agent, print its name and a one-line declared-slugs summary>

## Errors

<list of errors, or "None">

## Warnings (orphan fields)

<list of warnings, or "None">

## Coverage

<for each non-advisory field in each template:>
- <namespace>.<slug> ← [agent_a, agent_b]
or
- <namespace>.<slug> ← (orphan)

## Result

PASS  (no errors)
or
FAIL  (<N> errors)
```

### 7. Exit Behaviour

End the response with a single line:

> `Result: PASS` (no errors -- warnings allowed)

or

> `Result: FAIL (<N> errors)` (errors present)

If FAIL, also list a one-line remediation hint per error category at the end (e.g., "Add the missing field to the template, or remove the declaration from the agent.").

## Notes

- This validator is a static parse. It does not check whether the body of the agent prompt actually references the fields it declares -- that is left to human review.
- Advisory-only fields are intentionally excluded from orphan warnings. Use the marker for fields that exist for human reference (e.g., section purpose-of-section descriptions) and are not directly consumed by an agent.
