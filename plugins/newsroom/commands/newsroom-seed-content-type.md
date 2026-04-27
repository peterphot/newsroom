---
description: Create a new content type definition from a Socratic interview about structure, conventions, and resolution style
argument-hint: <name>
allowed-tools: Read, Write, AskUserQuestion
user-invocable: true
---

# Seed Content Type

Create a new content type definition. Content types are structural contracts: the Architect builds briefs from them, the Journalist writes to them, the Editor judges against them. The seeder runs a Socratic interview, not a form fill -- if the user cannot describe a section's purpose without using the section's name, push for the underlying job.

## Process

### 1. Get the Content Type Name

The name is provided as the argument. If absent, use `AskUserQuestion`:

> "What name would you like to give this content type? Use a short slug (lowercase, hyphens for spaces) -- this becomes the filename. Examples: `trade-media-article`, `customer-case-study`, `analyst-briefing`."

Normalise to lowercase with hyphens.

### 2. Check for Existing Definition

Use `Read` to check whether `newsroom/content-types/{name}.md` already exists. If it does, use `AskUserQuestion`:

> "A content type for '{name}' already exists at `newsroom/content-types/{name}.md`. Overwriting will discard the current definition. Options: [Overwrite] [Switch to /newsroom-refine-content-type on the existing file] [Cancel]"

Handle the choice as the publication seeder does.

### 3. Run the Socratic Interview

Use `AskUserQuestion` to work through the fields below. Discipline: every section must have a stated **purpose** distinct from its name. If the user says "the Hook section is for the hook", push: "what is the hook doing -- what does the reader feel or know after reading just it?" If the user lists sections that overlap, surface the overlap and ask which one owns the responsibility.

Capture:

#### Required fields

1. **One-line description** -- What this content type is for. Example: "A long-form opinion or analysis piece for an industry-specific audience that values evidence over narrative." Push back on definitions that could describe any article.

2. **Target word count and tolerance** -- A range, e.g. 800-1500 words. Ask why that range -- if the user cannot say, suggest tying it to the structural template's section budgets.

3. **Structural template** -- An ordered list of named sections. For each section, capture:
   - **Name**
   - **Purpose** (one sentence -- what job does this section do?)
   - **Approximate proportion** of total word count (e.g., 10-15%)
   - **What makes it good** (2-4 bullets)
   - **What makes it bad** (2-4 bullets)
   - **Optional:** example transition phrases or opening patterns

   The Architect uses this exact structure to build briefs, so be specific. Reference `${CLAUDE_PLUGIN_ROOT}/references/trade-media-article.md` for the level of detail required -- match it.

4. **Headline conventions** -- Case (sentence vs title), length range, patterns to favour, patterns to avoid. Ask for one acceptable headline and one unacceptable one.

5. **Lede style** -- Distinct from the structural Hook/Lede section: the *patterns* this content type favours and avoids in the opening. Examples of preferred lede patterns (data-led, event-led, question-led, scene-setting, etc.) and patterns to avoid.

6. **Citation and attribution conventions** -- How sources are cited (inline links, footnotes, parenthetical), what level of attribution is required (source name, date, methodology), and how quotes are introduced (full name, title, organisation on first reference?).

7. **Visual callouts** -- Which callouts the type supports (pull quotes, data callouts, sidebars, charts, photo captions, etc.). For each, capture: when to use, length / format, and any rules about frequency.

8. **Resolution style** -- How the piece resolves at the very end. Offer the user the standard options (CTA, takeaway, open question) and ask which apply -- and crucially, whether the type permits **mixed** resolutions or enforces a single one. The Architect records the chosen resolution in the brief and the Journalist commits to it; if the type permits mixing, say so explicitly.

#### Optional but useful

9. **Tone guidance** -- General tone characteristics for the type (formal/informal, technical depth, voice register).

10. **What distinguishes this type from adjacent types** -- E.g., "Trade media article vs general business journalism: assumes domain expertise, prioritises practical implication over narrative." This helps the Architect and Editor avoid mode-confusion.

### 4. Generate the Content Type Definition

Write `newsroom/content-types/{name}.md` using the structure of `${CLAUDE_PLUGIN_ROOT}/references/trade-media-article.md`. The file must include:

```
---
name: {name}
type: content-type
---

# {Display Name}

## Overview

**Typical word count:** {range}

---

## Structural Template

### 1. {Section Name}
**Purpose:**
**Typical length:**
**What makes it good:**
**What makes it bad:**
**Example patterns / transitions:** (optional)

(repeat for each section)

---

## Conventions

### Headline Conventions
### Lede Style
### Visual Callouts
### Resolution Style
### Attribution Style
### Data Presentation
### Quote Integration
### Tone Guidance
### What Distinguishes This Type
```

For any optional section the user did not provide, write a one-line placeholder: `_Not specified during seeding -- refine later with `/newsroom-refine-content-type`._`

### 5. Maintaining the Contract

The plugin enforces a static contract between bundled templates (`${CLAUDE_PLUGIN_ROOT}/references/`) and agent prompts (`${CLAUDE_PLUGIN_ROOT}/agents/`). The validator (`/newsroom-validate`) parses the bundled templates and checks that every section is consumed by at least one agent (or marked advisory-only).

You are writing a **user file** at `newsroom/content-types/{name}.md`, not the bundled template. The validator does not parse user files. However:

- **Stay within the bundled template's convention structure.** Agents only know how to read conventions that exist in the bundled `trade-media-article.md` (Headline Conventions, Lede Style, Visual Callouts, Resolution Style, Attribution Style, Data Presentation, Quote Integration, Tone Guidance). The structural-template sections themselves are content-type-specific and are consumed as a whole via `structural_template`, so you are free to define whatever named sections the user describes -- but invented top-level conventions outside the bundled set will not be read by any agent.
- **If the user insists on capturing a convention outside the bundled set**, add it under `## Conventions` with `<!-- advisory-only -->` on the line immediately after the heading. This signals to future maintainers that the field is informational only.
- **Extending the bundled template is a separate, developer-level change.** It requires updating the `<inputs>` declaration of the consuming agent(s) in the same change, then running `/newsroom-validate` to confirm. Do not touch `${CLAUDE_PLUGIN_ROOT}/references/trade-media-article.md` from within the seeder flow.

### 6. Confirm

> "Content type definition created at `newsroom/content-types/{name}.md`. The Architect will read this when building briefs that use this type. Refine it with `/newsroom-refine-content-type {name}` or list all definitions with `/newsroom-list`."
