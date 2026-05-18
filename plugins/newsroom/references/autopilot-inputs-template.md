# Autopilot Inputs

This is the input file for `/newsroom-auto`. Fill in each section and pass the file path with `--inputs <path>`, or let the interactive mode walk you through it (the interactive mode produces a file in this exact shape under the hood).

The autopilot reads four sections: `## Headline` (optional), `## Key Points` (required), `## Quotes` (required), and `## Transcript` (required). Section headings must match exactly. Anything outside these sections is ignored.

---

## Headline

<!-- Optional. A working headline or a short angle one-liner. The headline you want reflected in the final piece. If you leave this blank, the Architect will derive one from your key points. -->

## Key Points

<!-- Required. 3–7 bullets capturing the lead angle and the supporting threads. The first bullet should be the angle that the headline should reflect. Each bullet should be one or two sentences. Specific > generic. -->

- The lead angle that should drive the headline.
- A supporting thread (a counterpoint, a market trend, a specific implication).
- Another supporting thread.
- (Optional) A counter-argument or risk to address.
- (Optional) A "so what" — the desired takeaway.

## Quotes

<!-- Required. At least one quote. These are killer quotes that MUST appear in the final piece. Each quote MUST be a verbatim extract from the transcript below — if you have lightly edited for clarity, the autopilot will soft-warn but continue. Format: blockquote followed by an attribution line starting with em-dash or hyphen-dash. -->

> "Verbatim quote text goes here. Keep it punchy."
> — Name, Title, Company

> "A second killer quote, if you have one."
> — Name, Title, Company

## Transcript

<!-- Required. A timecoded, name-labelled transcript. Standard format: [HH:MM:SS] Name: text on one line, or paragraph form below. Any format works as long as speakers are labelled and timestamps are present. The autopilot treats this as the canonical evidence base — every factual claim in the article should be traceable to something in here. -->

[00:00:00] Interviewer: Thanks for making time. Let's start with the headline question — what's actually changing in the market?

[00:00:18] Speaker Name: Here's the thing most people miss...

[00:01:42] Interviewer: And the counter to that view is...?

[00:01:55] Speaker Name: ...

<!--
Notes for the user:

- Soft-warn behaviour: if a quote in the Quotes section doesn't appear verbatim in the Transcript, autopilot continues but logs the mismatch. The Fact Checker will surface it in the final report.
- Hard halt: if a required section is missing or empty, autopilot stops before any spend.
- Optional research: pass `--research quick` to add a tight data + industry sweep on top of the transcript. Off by default.
- Final review: a final-approval gate is on by default. Pass `--no-review` for a fully hands-off run.
-->
