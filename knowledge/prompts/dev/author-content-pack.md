# Prompt: Author or revise a content pack

Use when adding a new `modules/track-<a|b>-NN-<slug>.md` (or `modules/shared-NN-<slug>.md`) or revising one.

```
Author the content pack modules/track-<a|b>-<NN>-<slug>.md covering M<a>–M<b>.

Inputs (read in full first):
- knowledge/patterns/module-content-structure.md   (section order + rules)
- knowledge/rules/coding-standards.md               (naming, pins, snippet rules)
- knowledge/rules/arch-summary.md                   (amendments to ARCH v6.0)
- knowledge/references/aws-containers-sources.md    (the ONLY citable URLs + pinned versions)
- docs/curriculum/ecs-eks-curriculum.md             (module's Theory / Guided / Unguided text)
- the previous content pack in the same track       (continuity of names and stacks)

Mapping:
- Theory Sessions      -> each module's How It Works / Concepts
- Guided Activity      -> "## Hands-on lab" section for that module (full snippets)
- Unguided Challenge   -> "## Lab exercise" (spec + acceptance checks, NO solution)

Must:
- The pack is the learner copy: no "Program decision" callouts, no syllabus/deviation/approval talk, no budget figures. Record deviations only in knowledge/rules/arch-summary.md.
- Match the voice of the local-phase guides (https://github.com/stratpoint-engineering/devops-capstone-3tier-app/tree/main/docs) — see coding-standards.md "Prose — voice".
- Include at least one failure-path check in the Checkpoint.
- No "WSL2 vs macOS" section; put any platform-specific tip inline where it is needed (lab step comment or troubleshooting row).
- Never reference docs/, knowledge/, or ADR IDs in the output file.

Then run the quality gates listed in AGENTS.md and report results.
```
