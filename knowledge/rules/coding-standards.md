# Coding & Authoring Standards — DCTA-AWS Curriculum

Applies to every snippet (HCL, YAML, shell, Dockerfile, JSON) that appears
in `modules/*.md` and `capstone/*.md`, and to the prose around them. The
structural rules for content packs live in
`knowledge/patterns/module-content-structure.md` — this file adds the
technical conventions.

## Naming (used identically in every module)

| Thing | Name |
|---|---|
| Resource prefix | `dcta` |
| State keys | `dcta/<stack>.tfstate` (e.g. `dcta/10-foundation.tfstate`) |
| Security groups | `dcta-alb-sg`, `dcta-app-sg`, `dcta-rds-sg` |
| ECR repositories | `dcta-frontend`, `dcta-backend` |
| DB secret | `dcta/db/credentials` (JSON: `username`, `password`) |
| SSM parameters | `/dcta/db/host`, `/dcta/db/name` |
| RDS identifier | `dcta-postgres`, DB name `taskdb` |
| ECS | cluster `dcta-ecs`, services `dcta-frontend`, `dcta-backend` |
| EKS | cluster `dcta-eks`, app namespace `task-app` |
| Learner repo dirs | `app/`, `infra/<stack>/`, `deploy/` (Track B). No wrapper scripts: sessions use explicit `terraform -chdir=infra/<stack> apply` / `destroy` |

## Terraform

- `required_version = ">= 1.11"` (write-only arguments keep the DB password out of state); providers pinned **exactly** to the
  versions in `knowledge/references/aws-containers-sources.md`
  (`version = "6.66.0"`, not a range). Commit `.terraform.lock.hcl`.
- Backend: `backend "s3" {}` with partial config from `infra/backend.hcl`
  and `use_lockfile = true`, `encrypt = true`. Never local state.
- Provider `default_tags`: `ProjectCode = "Terraform101-CloudIntern"`, `Environment = "sandbox"`,
  `Owner = var.learner_id`, `CostCenter = "training"`, `ManagedBy = "terraform"`,
  `Stack = "<stack-dir>"`.
- `region = "ap-southeast-1"` only — via variable default, never per-resource.
- No hardcoded account IDs — use `data "aws_caller_identity"`.
- Any `0.0.0.0/0` must carry a `# justification:` comment. The only allowed
  case is **egress** from app-tier/nodes (needed to reach ECR/AWS APIs
  without NAT). Ingress is always `var.learner_cidr` or an SG reference.
- Cross-stack values come from `data "terraform_remote_state"`, never
  copy-pasted IDs.
- Outputs holding secrets: `sensitive = true`. Prefer not outputting them.
- All HCL in docs must pass `terraform fmt -check`.

## Helm provider 3.x syntax (common trap)

```hcl
provider "helm" {
  kubernetes = {
    host                   = data.aws_eks_cluster.this.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.this.certificate_authority[0].data)
    exec = {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args        = ["eks", "get-token", "--cluster-name", "dcta-eks", "--region", "ap-southeast-1"]
    }
  }
}
```

`set` is a list of objects: `set = [{ name = "...", value = "..." }]`.

## Kubernetes / Helm YAML

- Explicit `namespace:` on every namespaced object.
- `resources.requests` and `limits` on every app container (the single
  node is the budget).
- ESO objects use `apiVersion: external-secrets.io/v1`.
- Ingress uses `ingressClassName: nginx` (F5 NGINX Ingress Controller).
- YAML in docs must parse (`python3 -c "import yaml,sys; list(yaml.safe_load_all(sys.stdin))"`).

## Containers

- Base images from `public.ecr.aws/docker/library/…` with a version tag
  (`node:24-alpine`, `nginx:1.30-alpine`, `postgres:17-alpine`). Never `latest`.
- Images pushed with the Git short SHA as tag; never `latest`.
- Build for `linux/amd64` explicitly (Apple Silicon default is arm64).

## CI (gitlab.com)

- `image: docker:29` + `services: [docker:29-dind]` (as in GitLab's Docker-in-Docker docs).
- No AWS access keys in CI: jobs log in with GitLab OIDC (`id_tokens`, `aud: https://gitlab.com`) to the `dcta-gitlab-ci` role, whose trust is pinned to the project's `main` branch. `AWS_ROLE_ARN` is a protected (not secret) variable.
- **Every AWS resource is created by Terraform.** Labs never create AWS resources in the console or with the CLI (reading/inspecting is fine; one-off `run-task` jobs that stop by themselves are fine).
- Set `AWS_DEFAULT_REGION: ap-southeast-1` at pipeline level.
- No vulnerability-scan stage (out of scope, ADR E004-02).

## Prose — voice and readability (match the local-phase guides)

Trainees came from the local-phase guides in
[`devops-capstone-3tier-app/docs`](https://github.com/stratpoint-engineering/devops-capstone-3tier-app/tree/main/docs); keep the same voice so
the course feels continuous. Inside the required template sections:

- Friendly second person: "In this pack we'll…", "You'll…", "Don't worry if…".
  Short sentences. One idea per paragraph. Explain a term the first time it
  appears, in plain words.
- Each pack's **Objective** opens with a "What we're building" `mermaid`
  diagram (keep it to 4–8 nodes).
- Each module's **Core Idea** includes a **"Think of it like this:"**
  numbered list or everyday analogy.
- **Hands-on lab** steps are numbered `#### 1. …` headings, each with one
  sentence on *why*, then a command block where every command has a
  `# comment` line saying what it does, then **"You should see:"** with the
  expected result.
- End each Hands-on lab with an **"If something goes wrong"** table
  (symptom → likely cause → fix).
- Prefer short bullet lists to long tables; keep tables to ≤ 4 columns.
- Callouts: `> **Tip:**` (helpful shortcuts) only. No budget amounts.

## Prose — rules

- Commands in fenced blocks, one command per logical step.
- **`modules/*.md` is the learner copy.** Teach what the course uses as
  the way it's done — never mention the original syllabus, deviations,
  pending approvals, retired/rejected alternatives, or why the program
  chose one option over another. Those justifications live only in
  `knowledge/rules/arch-summary.md` and the instructor capstone. Technical
  "why" that helps the learner operate the system (e.g. "no NAT Gateway, so
  containers need public IPs to reach ECR") is fine.
- **Never state the program budget ($5 ceiling) or any dollar figure in
  `modules/*.md` or `capstone/*-cohort.md`.** The budget drives sizing
  decisions (authors/instructors only); trainees get general cost
  reasoning ("bills every hour it exists") instead.
- For cost context, link the AWS pricing page rather than quoting a rate.
  Instructor-only docs may quote planning rates, marked "verify on the
  pricing page".
