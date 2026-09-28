# Prompt: Author or revise a capstone pair

Use when changing `capstone/capstone-track-*-cohort.md` /
`capstone/capstone-track-*-instructor.md` or the rubric.

```
Update the Track <A|B> capstone pair.

Inputs (read in full first):
- knowledge/patterns/capstone-documentation-template.md   (section order, cohort vs instructor split)
- knowledge/patterns/technical-grading-rubric.md          (weights, documentation gate)
- knowledge/rules/arch-summary.md                          (amendments)
- every modules/track-<a|b>-NN-*.md, modules/track-<a|b>-capstone-requirements.md and modules/shared-01-aws-foundation.md
- docs/arch-docs/ARCH-DCTA-AWS-v6.0.md §7.2 (capstone topology)

Rules:
- Requirements must trace one-for-one to module Lab exercises; do not invent new ones.
- Technology stack rows must each map to a module the track actually teaches.
- Cohort version: no solution snippets, no Local setup, no Repository structure,
  no pasted outputs, Checkpoint items without answers.
- Instructor version: same sections + how each requirement is satisfied,
  key snippets, expected outputs, Checkpoint answers.
- Data source is the reference app's app/database/init.sql (decided 2026-09-28) — do not re-ask.
- Rubric: Tier 1 excluded -> 45 / 35 / 20 + pass/fail Documentation Gate.

Then run the quality gates in AGENTS.md.
```
