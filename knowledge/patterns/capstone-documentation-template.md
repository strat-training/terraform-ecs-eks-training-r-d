# Pattern: Capstone Documentation Template

A documentation template specifically for capstone deliverables — the
end-to-end project a cohort builds by integrating several modules'
skills into one running system. Use this instead of
`project-documentation-template.md` when the deliverable is a capstone;
that sibling template stays the generic pattern for non-capstone
project write-ups (dashboards, reports, apps).

This template produces two variants of the same document:
- **Cohort version** — everything except the solution itself (no full
  script/code, no filled-in Checkpoint answers). What a trainee gets as
  the spec to build against.
- **Instructor version** — the same document with the solution baked in
  (the actual script, the expected output, filled Checkpoint answers).

Fill in one Markdown file per variant from the sections below; do not
merge both into a single file with hidden/collapsed sections. The
current capstone's filled-in output lives in `capstone/` at the repo
root — `capstone/capstone-end-to-end-pipeline-instructor.md` and
`capstone/capstone-end-to-end-pipeline-cohort.md`, with shared test
fixtures in `capstone/mock-data/`. Use those two files together as the
worked example of this template; `modules/capstone-end-to-end-pipeline.md`
no longer exists — it was superseded by this split.

---

# [Capstone Name]

_One or two sentences: what this capstone builds, and which modules it
integrates._

> **Disclaimer** _(delete this callout if not applicable)_
>
> _State plainly if the scenario, company, or stakeholder is fictional/mock,
> and what real program or course this was built for._

## Getting started

_Placed first — before the problem framing below — because it gives the
reader the environment/context they need before anything else makes
sense._

### Prerequisites

_Tools/versions needed before setup._

### Local setup

_(Typically instructor-only — see the note at the end of this
subsection.) Copy-pasteable commands, in order, from clone to running.
End with a verification step: a command (or short sequence) the reader
runs to confirm it actually works — ideally proving both the happy path
and at least one failure path, matching "Validation & testing" below._

```bash
git clone <repository-url>
cd <project-directory>
# setup commands here
```

**Instructor version:** the setup commands may include the full solution
script. **Cohort version:** typically omits this subsection entirely,
keeping the doc as pure spec (what to build and why) rather than a build
guide. See the current capstone's two files —
`capstone/capstone-end-to-end-pipeline-cohort.md` vs.
`capstone/capstone-end-to-end-pipeline-instructor.md` — for the worked
precedent: the instructor version keeps this subsection because it lets
an instructor actually walk the build while validating a submission; the
cohort version drops it so it isn't handed back as a build guide.

### Data setup

_If the project needs data that isn't in the repo, say how to get it. If
a dataset/mock-data decision was already made in "Data sources" below,
point back to it here rather than re-deciding._

## Business problem

_State the single core question this project answers, as a blockquote._

_Then 2-4 bullets: what does this project help its intended user do?_

## Requirements / acceptance criteria

_Numbered list of the concrete, testable requirements this build must
satisfy — checkable conditions, not aspirational goals. Pull these from
the capstone's own spec/module file (e.g. `modules/
capstone-end-to-end-pipeline.md`'s "Requirements" section); don't invent
new ones or drop existing ones. Include at least one requirement that
names how failure is proven, not just success — see "Validation &
testing" below._

**Instructor version only:** note next to each requirement how the
solution satisfies it. **Cohort version:** list the requirement only,
with no hint at the implementation.

## Deliverable

_What is the final output — a dashboard, a report, an app, an API, a
pipeline? Pick whichever of the two modes below applies and delete the
other._

_**UI deliverable** (dashboard, app, report): describe each page/view/
screen with a one-line description and an image placeholder._

### [Page/View Name]

_What this page/view shows and who uses it._

![Description](path/to/screenshot.png)

_**Non-UI / automation deliverable** (a pipeline, script, or service with
no screen): describe its observable output instead — the log line
format, where processed/valid vs. rejected/invalid output ends up, and
how a scheduled run is confirmed (the cron entry used, the log line that
proves it fired on schedule rather than only when triggered by hand)._

## Data sources

_List each data source used. For each: where it comes from, and whether
it's a static/bounded input or a live/queried source._

> **Before filling this in, always ask the user which of three paths to
> take — never assume:**
> 1. A specific dataset/test case the user already has and wants
>    referenced in this doc.
> 2. A small mock/synthetic dataset the agent generates — just enough to
>    exercise the pipeline (e.g. a handful of valid records plus at
>    least one deliberately malformed one), not a large or realistic
>    dataset.
> 3. Skip data sourcing entirely for this doc (e.g. the trainee supplies
>    their own at build time, or it isn't applicable yet).
>
> This decision also settles what "Validation & testing" below can
> honestly claim was tested — don't ask a second time there, just point
> back to what was decided here.

## Project architecture

_Describe the data/processing flow, end to end. A simple text-fenced
diagram is enough — it doesn't need to be a rendered image._

```text
Source → Step → Step → Output
```

_Name what orchestrates this flow, if anything (a scheduler, a script, a
person running steps manually)._

## Data model

_If this project has a structured data model (dimensional, relational,
etc.), describe it here: entities, relationships, a schema diagram.
Delete this section if not applicable — a flat-file routing pipeline
(e.g. CSV in, processed/rejected out) typically has no data model to
describe._

## Technology stack

_Table of the actual tools this capstone uses. Every row must trace to a
module the cohort has actually completed — check `modules/` and the
architecture doc's capstone section (`docs/arch-docs/
linux-training.md` §5) before writing a row. Do not include a row for a
tool the course hasn't taught yet (e.g. Docker, Kubernetes, or a
separate observability tool) just because a fuller future version of
this capstone might use it — a mismatch between this doc and what was
actually built is worse than a short-looking table. This section keeps
the doc aligned to the training as delivered, so it must be re-checked
any time the capstone's module scope changes._

| Area | Technology | Module |
|---|---|---|
| _e.g. Scripting_ | | _e.g. M9_ |
| _e.g. Scheduling_ | | _e.g. M8_ |
| _e.g. Credential handling_ | | _e.g. M13_ |

## Key engineering features

_Bullet list of the technical decisions worth calling out — what makes
this implementation solid, not just "it works." **Instructor version:**
name the actual technique (e.g. "idempotent via `mv` draining
`incoming/`"). **Cohort version:** name the property required without
revealing the technique (e.g. "the script must be idempotent")._

## Validation & testing

_Describe how the deliverable was actually proven to work — not just
"it works," but what was run to confirm it. At minimum: the happy-path
run and its output, and at least one deliberate failure-path run showing
the failure is handled correctly (not just that it doesn't happen to
error). Quote the actual log/output lines as evidence where possible._

**Instructor version:** include the real log output from both runs.
**Cohort version:** describe what must be run and what a correct result
looks like, without pasting the actual log output as an answer key.

## Output & usage notes

_Where does the final artifact live, and how is it consumed? Include any
caveats a user needs to correctly interpret the output — what it does and
doesn't mean._

## Platform notes

_(Optional — delete this section if the project behaves identically
everywhere it runs.) A table of any behavior that actually differs by
local environment — OS-specific paths, cron/scheduler quirks, shell
differences. Only include rows where behavior genuinely diverges; don't
pad it with identical rows._

| Behavior | Environment A | Environment B |
|---|---|---|

## Repository structure

_(Typically instructor-only, for the same reason as "Local setup" above
— it helps an instructor validate a submission against the real layout.
The cohort version typically omits this section too.)_

```text
project-name/
├── ...   one-line purpose per top-level folder
```

## Documentation

_Links to deeper docs elsewhere in the repo, if any exist._

## Checkpoint (self-assessed)

_(Optional — delete this section for a non-training deliverable.) A
short checklist the trainee ticks off themselves confirming the build is
complete and correct — not a graded quiz. Mirror "Requirements /
acceptance criteria" above one-for-one so nothing is checked that wasn't
required._

- [ ] ...

**Instructor version:** may include expected answers/outputs alongside
each item. **Cohort version:** the checklist items only, self-assessed
by the trainee with no answers supplied.

## Project scope

_What this project does, and — just as important — what it explicitly
does NOT do. Name the real-world factors or caveats a reader needs before
acting on this project's output._
