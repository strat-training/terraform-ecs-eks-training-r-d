# Pattern: Repository README Structure

The root `README.md` of a training-curriculum repository follows this
section structure. It is the front door for **everyone** — learners,
instructors and course authors — so it orients, links out, and stays
short; the detail lives in the content packs, capstone files and
`knowledge/`. See this repo's own `README.md` for a filled-in example
(a two-track program: a shared foundation, then Track A or Track B).

````markdown
# <Program Phase Title>: <Main Technologies>

<2–3 sentence intro: what this repo contains, which phase of which
program it is, what it continues from (link), and what the learner
builds.>

## Project Overview

### Application Architecture
- **<Tier>**: <technology> (<where it runs>)

### What You'll Build

```mermaid
graph TD
    <foundation> --> <choice or next stage> --> <tracks/stages> --> <capstone>
```

- **<Shared part / stage> (M<n>):** <one line>
- **<Track or stage name> (M<a>–M<b>):** <one line>

<One sentence on the constant operating rules — region, daily routine, etc.>

### How It All Connects

```mermaid
sequenceDiagram
    <participants>
    <one deploy + one request, end to end; `alt` blocks only where tracks differ>
```

More detail for each track is in its capstone brief: [<Track>](./modules/<...>-capstone-requirements.md)

## Quick Start

1. **<Prerequisite phase first.>** <what is reused>
2. **<Get access / credentials>** <from whom>
3. **Start with <first pack>:** [<path>](./modules/<first-pack>.md)
4. **Pick your track** and read its capstone brief: <links>
5. **Work through your track's packs in order** <how order is shown>,
   tracking progress in [<track task list>](./dev-tasks/<...>.csv)

Tools you'll need: <list, with per-track extras marked>

```bash
# Check your tools
<one version command per tool>
```

## Project Structure

```text
.
├── modules/      # <annotated, one line per file>
├── dev-tasks/    # <learner task backlogs, one CSV per track>
├── capstone/     # <…>
├── docs/         # <…>
├── knowledge/    # <…>
└── AGENTS.md     # <…>
```

<One sentence restating the content-pack section order.>

## Implementation Requirements

### <Shared part> (M<n>)
- <one line per required capability>

### <Track / stage> 
- <capability> (M<n>)

## Documentation

- [<Label>](<relative path or verified URL>)

## Timeline

### Week <N>: <Theme>
- Days <a>-<b>: <what> (M<n>)

<One sentence pointing to the day-by-day plan.>

## Evaluation Criteria

<One line naming the rubric, linked.>

| Phase | Weight |
|---|---|
| <phase> | <n>% |

<Any pass/fail gate, one or two sentences.>

- **<Band>:** <range>

## For Course Authors

- Read [`AGENTS.md`](./AGENTS.md) first — <why>
- <pattern / rules / references files, linked, one line each>

## Support

Need help?
1. <first place to look>
2. <…>

Remember:
- <3–5 habits the learner must keep>
````

## Section Definitions (Reusable Format)

- **Title + intro:** `# <Phase>: <Main Technologies>`, then 2–3 sentences:
  what the repo is, which program/phase, what it continues (always a
  link — to the prerequisite repo, never a local copy that can go stale),
  and what the learner ends up building. Same friendly voice as the
  content packs.
- **Project Overview**
  - **Application Architecture:** one bullet per tier —
    `**<Tier>**: <tech> (<where it runs>)`. Only what the learner deploys.
  - **What You'll Build:** a `mermaid` `graph TD` of the learning path
    (foundation → choice/stages → capstone), **4–8 nodes**, then one bullet
    per stage/track with its module range (`M2–M8`). Close with a single
    sentence on constant rules (region, daily routine).
  - **How It All Connects:** a simple `sequenceDiagram` — one deploy and
    one request, end to end, **≤ 8 participants, ≤ 12 messages**. Give
    genuinely different components their own participant (e.g. ECS and EKS
    are separate boxes because they interact differently); use `alt`
    blocks only where tracks really diverge. Link to each track's capstone
    brief for the detailed per-track diagrams.
- **Quick Start:** 4–6 numbered, bolded action steps in the order a
  learner actually does them (prerequisite → access → first pack → pick
  track → work in order), then a tools sentence and one ```` ```bash ````
  block of version checks, each command commented, per-track tools marked.
- **Project Structure:** an annotated `text` tree of the real top-level
  folders and every learner-facing file, one `# comment` per line, marked
  "(instructors only)" where applicable. Follow the tree with one sentence
  restating the content-pack section order.
- **Implementation Requirements:** one `###` per shared part / track /
  stage, one bullet per capability the learner must deliver, each ending
  with its module number. Capabilities, not tasks — the task list lives in
  the capstone brief and `dev-tasks/`.
- **Documentation:** the handful of links a reader most needs (first pack,
  each track's brief and spec, rubric, prerequisite guides/repo). Relative
  paths for in-repo files; full URLs only for external repos.
- **Timeline:** one `###` per week, bullets as `Days a-b: <what> (M<n>)`;
  where tracks run in parallel, nest a bullet per track under the same
  days. One sentence pointing to the day-by-day plan (capstone briefs).
- **Evaluation Criteria:** name and link the rubric, a 2-column
  Phase | Weight table, any pass/fail gate in one or two sentences, then
  the score bands as bullets. Numbers must match the rubric file exactly.
- **For Course Authors:** the author-only entry points — `AGENTS.md`
  first, then the pattern, rules and references files, one line each, and
  a reminder that instructor files contain solutions.
- **Support:** a numbered "Need help?" list (troubleshooting tables, logs,
  lab sessions, supplemental reading) and a short **Remember:** list of
  3–5 habits (e.g. apply/destroy every session, never commit secrets,
  save evidence as you go).

## Rules

- **Everything is derived from the repo, not memory.** Module names and
  ranges come from `modules/`, weights and bands from the rubric file,
  the timeline from the capstone briefs, versions/tools from
  `knowledge/references/`. Re-check the README whenever any of those
  change (renames especially — file names appear in Quick Start, Project
  Structure and Documentation).
- **Learner-safe.** The README is read by learners: no program budget or
  dollar figures (the budget only guides capstone/infra decisions), no
  internal ADR IDs, no "why the program chose X over Y", no syllabus
  deviation talk. Those live in `knowledge/rules/` and instructor files.
  Linking to `AGENTS.md` and `knowledge/` in **For Course Authors** is
  fine — that section is explicitly for authors.
- **Links:** inline `[Label](url)` only, never `[Label]: url`. Every
  relative link must resolve (check with a script); every external URL
  must return HTTP 200 before it goes in.
- **Diagrams must render.** Validate every `mermaid` block (e.g. with
  `@mermaid-js/mermaid-cli`) and look at the output — clipped labels or
  overlapping arrows mean the diagram needs simplifying, not more nodes.
- **Keep it short and orienting.** No lab steps, no Terraform snippets, no
  per-module detail — link to where that lives. If a section grows past a
  screen, move the detail into the relevant pack or brief and link it.
- **Commands** go in fenced blocks with a `# comment` per command, same as
  the content packs.
- **Single-track programs:** drop the `alt` blocks, the "Pick your track"
  step and the per-track sub-bullets; keep every other section.
