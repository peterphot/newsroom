---
description: Create a new publication config from a Socratic interview about brand voice, audience, scope, and editorial standards
argument-hint: <name>
allowed-tools: Read, Write, WebFetch, AskUserQuestion
user-invocable: true
---

# Seed Publication

Create a new publication config by interviewing the user about the publication's voice, audience, mission, scope, and editorial standards. The output is a config file that downstream agents (Architect, Journalist, Editor, Fact Checker) read to steer the editorial pipeline. Generic answers in produce generic content out -- so the interview is Socratic, not a form fill. Push back on vague answers, ask for concrete examples, surface contradictions.

## Process

### 1. Get the Publication Name

The publication name is provided as the argument to this command. If no name was provided, use `AskUserQuestion` to ask:

> "What name would you like to give this publication? Use a short slug (lowercase, hyphens for spaces) -- this becomes the filename."

Normalise the slug deterministically:

1. Lowercase the input.
2. Replace each run of whitespace with a single `-`.
3. Strip every character outside `[a-z0-9-]` (drops apostrophes, ampersands, accents after step 1; transliterate accents only if the user asks).
4. Collapse repeated `-` runs to a single `-`.
5. Trim leading and trailing `-`.

Echo the resulting slug back to the user (`Using slug: my-pub`) and confirm before proceeding.

### 2. Check for Existing Config

Use `Read` to check whether `newsroom/publications/{name}.md` already exists. If it does, use `AskUserQuestion`:

> "A publication config for '{name}' already exists at `newsroom/publications/{name}.md`. Overwriting will discard the current config. Options: [Overwrite] [Switch to /newsroom-refine-publication on the existing file] [Cancel]"

If the user picks **Switch to refine**, tell them to run `/newsroom-refine-publication {name}` and stop. If **Cancel**, stop. If **Overwrite**, continue.

### 3. Gather Reference Material

Use `AskUserQuestion` to collect optional grounding material:

- **Reference article URLs** -- 2-4 published pieces that exemplify the voice. The seeder will fetch these and use them to ground the voice extraction (mirroring the journalist seeder).
- **Pasted excerpts** -- Optional. Text the user wants to anchor the voice on.
- **Existing brand guidelines** -- Optional. Pasted text or a URL.

Reference material is optional but strongly recommended. If the user provides nothing, note it and proceed -- the interview will need to work harder.

### 4. Fetch Reference Content

If URLs were provided, use `WebFetch` to retrieve each. If a fetch fails, retry once. If it still fails, note the failure and continue with what succeeded.

### 5. Run the Socratic Interview

Use `AskUserQuestion` to work through the fields below. The interview is **Socratic, not extractive**. For every answer:

- **Global gate:** before accepting any answer, ask yourself: could a competing publication give the same answer? If yes, push back with "what makes that distinctive to *this* publication?" Reject tone rules, voice examples, mission statements, or audience descriptions that any competent B2B publisher could have produced.
- If an answer is vague ("authoritative but accessible", "speaks to marketers", "not too corporate"), ask for a concrete example. Acceptable: a sentence in that voice. A specific reader the user has in mind. A piece they would never publish and why.
- If two answers contradict (e.g., "irreverent" tone but "Bloomberg-style" voice examples), surface the contradiction and ask which one wins.
- If the user gives a single example, ask for one more so a pattern can be seen.

Capture the following:

#### Required fields

1. **Brand name and parent organisation** -- Full brand name, and the parent org or company behind it (if any).

2. **Mission** -- One sentence: why this publication exists. Push back if the answer is "to share insights" or similar -- ask "for whom, doing what, that they could not do without you?"

3. **Target reader** -- Role, seniority, sector, and what they actually care about day-to-day. Push for specificity: "marketers" is not a reader; "performance marketing leaders at mid-to-large brands evaluating MMM solutions" is.

4. **Tone rules -- Always** -- At least 3 explicit "always" rules with one example sentence each.

5. **Tone rules -- Never** -- At least 3 explicit "never" rules with examples of the phrasing or pattern to avoid.

6. **Topical scope -- In** -- Topics this publication covers.

7. **Topical scope -- Out** -- Topics this publication explicitly does not cover. If the user says "we cover everything", push: "what would make a piece feel off-brand?"

8. **Voice examples** -- At least 2 sentences written in this publication's voice. If the user cannot produce them, draft 2-3 candidates from the reference material and ask them to pick / edit. The voice examples are the Journalist's anchors.

9. **Byline format** -- How authors are credited (e.g., "By [First Last], [Role] at [Brand]" -- or a generic byline if individuals are not named).

10. **Sign-off conventions** -- Whether pieces end with a sign-off ("Thanks for reading", contact line, etc.) or not.

11. **CTA conventions** -- What the closing call-to-action looks like. Single CTA or multiple? Sales-y or editorial? Integrated into the prose or appended?

12. **Distribution context** -- Where pieces run: own channel, trade media placement, syndication, or a mix. If a mix, ask for rough proportions and whether framing changes by channel.

13. **Required disclosures and compliance notes** -- Any commercial-relationship disclosures, regulated-topic rules, or legal review triggers the publication enforces. If the user says "none", probe: "if you cite a partner vendor, do you flag the relationship?" "Are forward-looking financial claims allowed?"

#### Optional but useful

14. **Content pillars** -- 3-6 topical pillars the publication organises content around.

15. **Terminology preferences** -- Preferred terms, terms to avoid, jargon policy. Mirror the structure in `${CLAUDE_PLUGIN_ROOT}/references/publication-template.md`.

16. **Style rules** -- Headings, numbers, dates, oxford comma, paragraph length, citation style, opening / closing patterns, emphasis usage.

17. **Google Docs target folder** -- Drive folder ID for final article output (optional).

### 6. Generate the Publication Config

Write `newsroom/publications/{name}.md` mirroring the exact section order of `${CLAUDE_PLUGIN_ROOT}/references/publication-template.md`. The file must include all sections in this order:

```
---
name: {name}
type: publication
---

# {Brand Name} Publication Config

## Mission

## Brand Voice

## Audience

## Tone Rules
### Always
### Never

## Topical Scope
### In scope
### Out of scope

## Voice Examples

## Content Pillars

## Terminology
### Preferred Terms
### Terms to Avoid
### Jargon Policy

## Past Content References

## Style Rules

## Byline, Sign-off, and CTA Conventions

## Distribution Context

## Required Disclosures and Compliance Notes

## Google Docs Output
```

**Required vs. optional sections:**

- **Required (must have real content):** Mission, Brand Voice, Audience, Tone Rules, Topical Scope, Voice Examples, Style Rules, Byline/Sign-off/CTA Conventions, Distribution Context, Required Disclosures. If a required field went unanswered or got a generic answer, **loop back to step 5 and re-ask** — do not write the placeholder string.
- **Optional (placeholder allowed):** Content Pillars, Terminology, Past Content References, Google Docs Output. For these, if the user provided nothing, write the one-line note: `_Not specified during seeding -- refine later with `/newsroom-refine-publication`._`

If the user provided reference URLs, list them under **Past Content References** with a one-line note on what each exemplifies.

### 7. Maintaining the Contract

The plugin enforces a static contract between bundled templates (`${CLAUDE_PLUGIN_ROOT}/references/`) and agent prompts (`${CLAUDE_PLUGIN_ROOT}/agents/`). The validator (`/newsroom-validate`) parses the bundled templates and checks that every section is consumed by at least one agent (or marked advisory-only).

You are writing a **user file** at `newsroom/publications/{name}.md`, not a bundled template. The validator does not parse user files. However:

- **Stay within the bundled template's section structure.** The agents only know how to read sections that exist in the bundled `publication-template.md`. If you invent a new section in a user file, no agent will read it.
- **If the user insists on capturing something outside the template structure**, append a new `##` section at the end of the file and add `<!-- advisory-only -->` on the line **immediately** after the heading — no blank line between heading and marker. The validator only recognises this exact placement. Example:

  ```markdown
  ## Custom Section Name
  <!-- advisory-only -->

  Content here...
  ```
- **Extending the bundled template is a separate, developer-level change.** It requires updating the `<inputs>` declaration of the consuming agent(s) in the same change, then running `/newsroom-validate` to confirm. Do not touch `${CLAUDE_PLUGIN_ROOT}/references/publication-template.md` from within the seeder flow.

### 8. Confirm

When the file is written, confirm:

> "Publication config created at `newsroom/publications/{name}.md`. You can use it in a session with `/newsroom --publication {name}`, refine it with `/newsroom-refine-publication {name}`, or list all configs with `/newsroom-list`."
