# Pattern: Content-Pack File Structure

Each content-pack file (`modules/track-<a|b>-NN-<topic-slug>.md`, or
`modules/shared-NN-<topic-slug>.md` for a pack both tracks use; `NN` is the
two-digit order within the track, continuing after the shared packs) follows this
section structure. See `modules/week-01-linux-fundamentals-cli-basics.md`
for a filled-in example.

```markdown
# Week N — <Topic> (M<start>–M<end>)

## Objective
## Topics
## M<n>: <Module Name>
- Learning Objective
- Core Idea
- Why It Matters
- How It Works
  - Concepts
  - Best Practices
  - Real-World Example
- Supplemental Reading
## Hands-on lab
### M<n> — <lab section, one per module in this pack>
## Lab exercise
## Checkpoint (self-assessed)
```

## Section Definitions (Reusable Format)

When filling out the `M<n>: <Module Name>` section, use a highly conversational, analogy-driven style. Break complex topics down into a clear "Problem/Solution" format. Use the following guidelines for any topic:

- **Learning Objective:** A welcoming, conversational statement of what the trainee will achieve. 
  *(Format: "This guide will help you create/understand [Topic] for [Context]. We'll break it down into simple steps.")*
- **Core Idea:** A high-level framing question. 
  *(Format: "What is [Topic] and Why Do We Need It?")*
- **Why It Matters:** Explicitly define the pain point and how the tool fixes it.
  *(Format: State the **Problem:** [e.g., "Many repetitive files"]. State the **Solution:** [e.g., "Tool Name - Template everything, package it up, easy to update"].)*
- **How It Works:**
  - **Concepts:** A simple bulleted list of what the technology actually does to solve the problem, including essential **terminologies**. 
    *(Format: "[Topic] helps us manage [System] by: [Action 1], [Action 2]. Key terms to know: **[Term 1]** (Simple definition), **[Term 2]** (Simple definition).")*
  - **Best Practices:** 1-2 actionable rules or guardrails for using this safely in the real world.
  - **Real-World Example:** A relatable analogy comparing the technology to universally understood tools from other ecosystems. 
    *(Format: "Think of it like: [Tool A] for [Ecosystem A], or [Tool B] for [Ecosystem B]. But for [Current Ecosystem]!")*
- **Supplemental Reading:** A curated list of official documentation or authorized research links formatted as `[Label](url)`. See the Rules below for strict sourcing requirements.

## Rules

- **Supplemental Reading** is populated only from real, cited sources.
  Check `knowledge/references/` first (the raw research
  material, with actual URLs attached to specific topics) — it's more
  authoritative than the arch doc's Resource Map (`docs/arch-docs/
  linux-training.md` §7), which only has titles, no URLs. If no source
  material exists for a topic, say so explicitly with a `**Note:**` line
  instead of writing Concepts/Best-Practices/Real-World-Example content
  from general knowledge dressed up as sourced material — see M2
  (`week-01`) for the zero-source precedent, and M4 (`week-02`) /
  M6/M7 (`week-03`/`week-04`) for the narrow-source precedent: cite the
  one real, specific source that exists and say explicitly what it does
  and doesn't cover, rather than treating a partial source as if it
  covered the whole topic.
- **Commands always go in fenced code blocks**, never as inline
  backticks strung together as if they were a runnable line — even a
  single command gets its own ```` ```bash ```` (or ```` ```powershell ````
  / ```` ```ini ````, as applicable) block. Inline backticks are for
  naming a command/flag/file/path in prose (e.g., "the `ls` command"),
  not for showing something the trainee is meant to type or paste.
- **Never reference this repo's own internal file paths in
  trainee-facing content.** Files under `modules/` are read by trainees,
  who have no access to this project's `docs/`, `knowledge/`, or
  `graphify-out/` directories. A "no source" `**Note:**` should say
  *"No cited source material exists for this topic in the research
  gathered for this course"* — never name where that was checked (e.g.
  never write "`knowledge/references/` has no entry for this"). This
  rule applies only to `modules/*.md`; internal docs like this pattern
  file or `knowledge/rules/arch-summary.md` may reference paths freely,
  since only course authors read those.
- **Link syntax:** always use inline links, `[Label](url)`. Never use
  `[Label]: url` (Markdown reference-*definition* syntax) — it only
  renders if something else in the document references `[Label]`
  elsewhere; used standalone, as a "citation," it silently disappears
  from the rendered page instead of showing as a link. (`week-01`'s M1
  citation still has this bug as of this writing — fix it opportunistically
  if you're ever editing that file for another reason.)
- Every content-pack file ends in a self-assessed Checkpoint — a short
  checklist the trainee ticks off themselves, not a graded quiz.
- Files are flat, one per content pack, directly under `modules/` — never
  nested in a per-module folder, never named `README.md`.