# AGENTS.md — DCTA-AWS Containerization in AWS Curriculum

## What this repo is

Curriculum source for **Containerization in AWS** (project code DCTA-AWS),
Weeks 5–6 of the DevOps bootcamp. It continues `docs/devops-bootcamp`
(Weeks 1–4, local Minikube + GitLab CI + Argo CD + observability) by taking
the same 3-tier task app to AWS on one of two tracks:

- **Shared (M1):** VPC, ECR, RDS PostgreSQL, Secrets Manager — Terraform.
- **Track A (M2–M8):** ECS on Fargate/Fargate Spot, ALB, GitLab CI, CloudWatch,
  ECS blue/green, IAM task roles.
- **Track B (M9–M16):** EKS, IRSA, External Secrets Operator, Helm,
  F5 NGINX Ingress, Argo CD, Prometheus/Grafana, HPA.

Hard constraints: `ap-southeast-1` only, one sandbox account per learner,
≤ $5.00 USD total spend, daily `terraform destroy`.

## Stack (content, not code)

Deliverables are Markdown. There is no application or runnable IaC tree in
this repo — Terraform/Helm/CI appears as snippets inside the docs.

| Path | Audience | Purpose |
|---|---|---|
| `modules/shared-01-aws-foundation.md`, `modules/track-[a,b]-NN-*.md` | Trainees | Content packs (one per group of modules; the shared M1 pack serves both tracks) |
| `modules/track-[a,b]-capstone-requirements.md` | Trainees | Capstone project brief per track (patterned on `docs/devops-bootcamp/.../docs/capstone-requirements.md`) |
| `capstone/*-cohort.md` | Trainees | Capstone spec — no solutions |
| `capstone/*-instructor.md` | Instructors | Same spec + solution snippets + answers |
| `capstone/capstone-grading-rubric.md` | Both | Weighted rubric + documentation gate |
| `README.md` | Everyone | Repo overview, structure, quick start, timeline, evaluation |
| `docs/arch-docs/` | Authors | ARCH v6.0 + ADRs (source of truth) |
| `docs/curriculum/` | Authors | Original syllabus |
| `docs/devops-bootcamp/` | Authors | Prerequisite phase + reference app |
| `knowledge/` | Authors / agents | Patterns, rules, references, prompts |

## Authoring standards

- Structure: `knowledge/patterns/module-content-structure.md` (content packs)
  and `knowledge/patterns/capstone-documentation-template.md` (capstones).
- Technical conventions: `knowledge/rules/coding-standards.md`.
- Decisions and amendments to ARCH v6.0: `knowledge/rules/arch-summary.md`.
- Sources: only URLs listed in `knowledge/references/aws-containers-sources.md`
  (each verified HTTP 200). Add and verify a URL there before citing it.
- Versions: only the pinned versions in that same file. Never write a
  version from memory; re-run `resolve_package_versions` to bump.

## Security rules (for every snippet)

- No real account IDs, access keys, passwords, or tokens — placeholders
  like `<ACCOUNT_ID>` only.
- Ingress restricted to `var.learner_cidr`; SG-to-SG references for app → DB.
- Secrets only via Secrets Manager → ECS `secrets` / ESO. Never in Git,
  `.tfvars`, ConfigMaps, or plain `environment`.
- Every Terraform snippet carries the standard `default_tags`.

## Trainee-facing hygiene (hard rules)

- `modules/*.md`, `capstone/*-cohort.md` and the rubric are the **learner
  copy**: never reference this repo's `docs/`, `knowledge/`, or
  `graphify-out/` paths, internal ADR IDs, the program budget, or why the
  program chose one option over another (that lives in
  `knowledge/rules/arch-summary.md` and the instructor capstones).
- Voice matches `docs/devops-bootcamp` (see coding standards).
- No wrapper scripts: labs show explicit `terraform -chdir=infra/<stack> apply|destroy`.
- Inline links only: `[Label](url)` — never `[Label]: url`.
- Commands always in fenced code blocks.

## Git conventions

- Conventional commits (`docs(modules): …`, `docs(capstone): …`, `chore(knowledge): …`).
- One content pack or capstone pair per commit where practical.

## Quality gates (run before calling content done)

1. Every ```` ```hcl ```` block passes `terraform fmt -check` (extract to a temp file).
2. Every ```` ```yaml ```` block parses with PyYAML.
3. Every Supplemental Reading URL returns HTTP 200.
4. Learner copy (`modules/`, `capstone/*-cohort.md`, rubric) has no internal
   paths, ADR IDs, budget figures, "Program decision" callouts, syllabus
   talk, or "WSL2 vs macOS" section. Check with
   `grep -rnE "(^|[^/])(docs|knowledge)/|ADR-DCTA|\$[0-9]|[Bb]udget|Program decision|syllabus|WSL2 vs macOS" modules/ capstone/*-cohort.md capstone/capstone-grading-rubric.md`
   (ignore matches inside URLs and "AWS Budgets").
5. `grep -rnE "^\[[^]]+\]: http" modules/ capstone/` returns nothing.
6. Instructor HCL snippets pass `terraform validate` when assembled into their stacks.
