# Architecture Summary — DCTA-AWS (Containerization in AWS)

Author-facing digest of `docs/arch-docs/ARCH-DCTA-AWS-v6.0.md` and its four
ADR files, **plus the course-authoring amendments made on 2026-09-28**.
When this file and the ARCH doc disagree, the amendments table below wins
for curriculum content; the ARCH doc itself has not been re-versioned.

## Shape

Self-paced 2-week continuation (Weeks 5–6) of `docs/devops-bootcamp`
(Weeks 1–4, local Minikube). Same reference app
(`devops-capstone-3tier-app`: React + Node/Express + PostgreSQL), same
gitlab.com project and pipeline. One shared module (M1), then learner picks
**Track A (ECS/Fargate, M2–M8)** or **Track B (EKS, M9–M16)**. Region
`ap-southeast-1`, one sandbox account per learner, **≤ $5.00 USD** total.

## Terraform stack topology (ADR E004-01)

| Stack | Lifecycle | Contents |
|---|---|---|
| `00-bootstrap` | Once, program-owned | S3 state bucket (native lockfile) — not learner work |
| `10-foundation` | Persistent 2 weeks | VPC (2 AZ, 2 public + 2 private, IGW, no NAT), SGs (`alb`, `app`, `rds`), ECR ×2 (keep last 2), Secrets Manager DB secret, SSM params, GitLab OIDC provider + `dcta-gitlab-ci` role (Track A) |
| `20-data` | Daily | RDS PostgreSQL 17 `db.t4g.micro`, private subnets, writes `/dcta/db/host` to SSM |
| `30-track-a` | Daily | ECS cluster, task defs, services, ALB, CloudWatch, autoscaling, blue/green, task roles, seed task |
| `40-track-b-cluster` | Daily | EKS 1.36 + 1 Spot `t3.medium` node, OIDC, IRSA role for ESO |
| `41-track-b-platform` | Daily | `helm_release`: ESO, F5 NGINX Ingress (`wait = true`, `allowEmptyIngressHost`), Argo CD. Monitoring stack + metrics-server are Argo CD Applications |

Teardown (ADR E004-03): `terraform destroy` per stack, reverse order —
run as explicit `terraform -chdir=infra/<stack> destroy` commands, **no
wrapper scripts** (user direction 2026-09-28; ARCH §7.4's `up.sh`/`down.sh`
are not used).
Track A: `30 → 20`. Track B: `41 → 40 → 20`. `10-foundation` destroyed only
on the last day.

## Amendments to ARCH v6.0 (course-authoring decisions, 2026-09-28)

| Topic | ARCH v6.0 / syllabus | Curriculum teaches | Why |
|---|---|---|---|
| Trivy (M4) | Syllabus has it; ARCH removed it | No scan stage | ADR E004-02 deprecated |
| Track B node | Syllabus `t3.small` | `t3.medium` Spot, flagged "pending sponsor approval", **plus VPC CNI prefix delegation** (max pods 110) | ADR E003-02 assumed 17 pods was enough; the full M15 stack needs ~20 pods, so the default CNI limit is still too low |
| M16 teardown | Syllabus `eksctl delete cluster` | `terraform destroy` only; `eksctl` diagnostics only | ADR E003-03 |
| Ingress controller | ARCH: ingress-nginx | **F5 NGINX Ingress Controller** (`nginx-ingress` chart), Service `type: LoadBalancer` + `aws-load-balancer-type: nlb` annotation | ingress-nginx retired March 2026 (user decision) |
| M6 blue/green | ARCH/PRD: AWS CodeDeploy | **ECS-native blue/green** (`deployment_configuration.strategy = "BLUE_GREEN"`) on the existing `/api/*` rule | CodeDeploy needs a dedicated listener per service; can't shift a path rule on a shared listener (user decision) |
| Grafana exposure | ARCH: `/grafana` path on the shared LB | Host rule `grafana.dcta.test` on the same NLB | F5 NIC allows only one host-less Ingress (the app's) |
| RDS SSL | Not mentioned | Backend connects with TLS using the RDS regional CA bundle | RDS PG 15+ defaults `rds.force_ssl=1` |
| DB seed | ARCH Open Q5 | Idempotent `seed.sql`, run daily by a one-off ECS task (A) or K8s Job (B) | App has no migrations; `init.sql` is not re-runnable |
| Frontend image | Dev-server Dockerfile | Multi-stage build → `nginx:1.30-alpine`, `REACT_APP_API_URL=/api`, no `proxy_pass http://backend` | Dev server OOMs on 0.5 GB; `backend` hostname doesn't resolve outside Compose |
| CI identity | ARCH §6: scoped IAM CI user (access key) | GitLab OIDC provider + `dcta-gitlab-ci` IAM role in `10-foundation`, trust pinned to `main` | An access key is created outside Terraform (untracked) and blocks deleting the user on the last day; OIDC keeps everything in Terraform with no long-lived key. Persistent so merges still push while `30-track-a` is down |

## Security rules

- Inbound only from the learner's `/32` (`TF_VAR_learner_cidr` exported at the start of each session).
- SG chain: `alb-sg`/EKS node SG → `app-sg` → `rds-sg:5432`. No CIDR rule on RDS.
- One Secrets Manager secret (`dcta/db/credentials`); everything else in SSM.
- Least privilege proven by a **denied** call (Track A M7; Track B M10).
- No secrets in Git, `.tfvars`, or Terraform outputs (`sensitive = true`).

## Data

Single store: RDS PostgreSQL, `tasks` table from the reference app's
`app/database/init.sql` (UUID PK, `status` CHECK in TODO/IN_PROGRESS/DONE).

## Rubric

Weighted, Tier 1 excluded (the capstone docs define requirements and
architecture): Technical Execution 45% / Functional Demo 35% / Defense 20%,
plus a pass/fail Technical Documentation Gate.
