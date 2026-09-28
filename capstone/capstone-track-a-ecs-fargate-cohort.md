# Capstone — Track A: Task Manager on ECS Fargate

In this capstone you bring together everything from M1–M7: you'll run the
task-manager app on **Amazon ECS with Fargate**, behind **one Application
Load Balancer**, deployed automatically by **GitLab CI** with
**blue/green** releases, watched by **CloudWatch**, scaled automatically,
and locked down with **least-privilege IAM** — all built with
**Terraform**, and all torn down at the end of every session.

> **Disclaimer**
>
> The team and scenario below are fictional. This capstone was written for
> the DevOps Bootcamp's *Containerization in AWS* phase and uses the same
> reference task-manager app (React, Node.js/Express, PostgreSQL) you
> deployed to Minikube in the local phase.

## Getting started

### Prerequisites

You need everything from M1–M7 working:

- Your AWS sandbox account (region `ap-southeast-1`) and your Terraform
  state bucket name.
- Your gitlab.com project from the local phase, with the app changes from
  M1 (TLS for the database, production frontend image, `seed.sql`).
- Tools: Terraform 1.11+, AWS CLI v2, Docker with `buildx`, `git`, `curl`,
  `jq`, `ab` (ApacheBench).
- The daily routine from M1: settings → `10-foundation` → `20-data` →
  `30-track-a` at the start; destroy in reverse order at the end.

### Data setup

The database is empty every session. The data comes from the reference
app's schema and sample rows (see **Data sources**) — your one-off seed
task loads them at the start of each session. There's nothing to
download.

## Business problem

> Can a small team run its task-manager app on AWS so that releases are
> safe and automatic, the app handles busy moments by itself, and nothing
> can reach the database or its password except the app — while the whole
> environment can be switched off at the end of every day?

This capstone helps the (fictional) TaskFlow team:

- release a backend change by merging to `main`, test it privately, and
  switch back instantly if something's wrong;
- stop worrying about busy hours — the backend adds tasks by itself;
- prove, not just claim, that every part of the app can only do what it needs;
- keep costs low by running infrastructure only while someone is working.

## Requirements / acceptance criteria

Each requirement comes from one module's Lab exercise.

**Foundation (M1)**

1. A VPC in `ap-southeast-1` across 2 AZs with **2 public and 2 private
   subnets**, an Internet Gateway, and **no NAT Gateway**.
2. Security groups `dcta-alb-sg` → `dcta-app-sg` → `dcta-rds-sg`: internet
   traffic only from your `/32`; RDS allows `5432` **only** from
   `dcta-app-sg`.
3. Terraform split into `10-foundation`, `20-data` and `30-track-a`, with
   remote S3 state (`use_lockfile = true`), exact provider versions, and
   stacks reading each other's outputs with `terraform_remote_state`.
4. ECR repositories `dcta-frontend` and `dcta-backend` with immutable tags,
   keeping the last 2 images; images are `linux/amd64` and tagged with a
   Git SHA.
5. The DB credentials are in Secrets Manager; the DB host and name are in
   Parameter Store. **No password in Git or in any Terraform state file.**
6. RDS PostgreSQL `db.t4g.micro` in the private subnets, not publicly
   accessible, and re-creatable every session with no manual steps.
   **Failure proof:** a connection from your laptop to RDS on `5432` fails.

**Compute and routing (M2–M3)**

7. Frontend and backend task definitions: Fargate, `X86_64`, 256 CPU /
   512 MiB, SHA-tagged images; the backend gets its DB settings only
   through `secrets` (nothing sensitive in `environment`).
8. Both services use **70% `FARGATE_SPOT` / 30% `FARGATE`**. A one-off seed
   task loads the database and exits `0`; running it again adds no rows.
9. One ALB: `/api/*` → backend, everything else → frontend.
   **Failure proof:** with the backend at 0 tasks, `/api/tasks` returns
   `503` while `/` returns `200`.

**Delivery (M4, M6)**

10. A merge to `main` builds and pushes both images and deploys them,
    using a CI identity that can only touch your repositories and
    services and logs in to AWS with GitLab OIDC (no access key); the
    deploy job fails if the service doesn't become stable.
11. The frontend deploys with zero downtime (rolling). **Failure proof:** a
    broken image makes the deploy fail and ECS roll back, with no errors
    for users.
12. The backend deploys **blue/green** with a header-based test route and a
    5-minute bake time. **Failure proof:** you roll back a deployment
    during bake time.

**Operations and security (M5, M7)**

13. Every container logs to a Terraform-managed CloudWatch log group with
    7 days' retention or less, and a CloudWatch dashboard shows service CPU
    and ALB requests/5xx.
14. The backend scales **1 → 3 tasks at 70% CPU** under load and back again,
    and the task list shows both Spot and regular Fargate.
15. Separate task roles: the backend may read only its own secret and two
    parameters; the frontend may do nothing. **Failure proof:** calls to
    another secret, `list-secrets` and `s3 ls` are denied.

**Defense and teardown (M8)**

16. A live, 15-minute defense following the rubric's agenda, ending with
    `terraform destroy` in reverse order and proof that no load balancer or
    NAT Gateway is left.

## Deliverable

This is a running system plus the evidence that it works — there's no
new user interface to design.

What the facilitator will look at:

- **Your app** at the ALB address: the task-manager UI on `/`, JSON on
  `/api/tasks`.
- **Your pipeline** on gitlab.com: build → deploy on `main`, green only
  when the service is stable.
- **CloudWatch:** log groups for every container, the `dcta-ecs` dashboard,
  and the backend's scaling activity.
- **Your repository:** `app/`, `infra/<stack>/`, `.gitlab-ci.yml`, and
  `docs/capstone/` with your evidence (see **Documentation**).

## Data sources

| Source | Where it comes from | Type |
|---|---|---|
| Task schema and 3 sample tasks | The reference app's `app/database/init.sql`, turned into the re-runnable `app/database/seed.sql` in M1 | Static, small |
| Tasks you create in the demo | Typed in through the app | Live, in RDS for one session |
| Failure-path inputs | A stopped backend, a broken image, a secret your role can't read | Deliberate, made up for tests |

## Project architecture

```text
Developer ──git push──► gitlab.com CI ──build/push──► ECR
                              │
                              └──register task def + update-service──► ECS
                                                                        │
Browser (your /32) ──► ALB :80 ─┬─ /api/* + X-Deploy-Stage: test ─► backend green (during deploy)
                                ├─ /api/* ───────────────────────► backend blue ⇄ green
                                └─ default ──────────────────────► frontend (rolling)
                                                                        │
                     backend ──TLS :5432──► RDS PostgreSQL (private subnets)
                     backend ◄── secrets at start ── Secrets Manager + Parameter Store
                     all tasks ──logs──► CloudWatch Logs ──► dashboard ◄── CPU metric ──► Auto Scaling (1–3)
```

What drives it:

- **You**, each session: `terraform apply` on `10-foundation` → `20-data` →
  `30-track-a`, then the seed task; `terraform destroy` in reverse order at
  the end.
- **GitLab CI**, on every merge to `main`: build, push, deploy.
- **ECS and Application Auto Scaling**, continuously: keep tasks healthy,
  switch blue/green traffic, scale on CPU.

## Data model

One table, `tasks`, from the reference app:

| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key, generated |
| `title` | VARCHAR(255) | Required |
| `description` | TEXT | Optional |
| `status` | VARCHAR(20) | `TODO`, `IN_PROGRESS` or `DONE` |
| `created_at`, `updated_at` | TIMESTAMPTZ | `updated_at` set by a trigger |

## Technology stack

| Area | Technology | Module |
|---|---|---|
| Infrastructure as code | Terraform 1.11+, AWS provider 6.66.0, S3 state with native locking | M1 |
| Network | VPC, public/private subnets, Internet Gateway, chained security groups | M1 |
| Registry | Amazon ECR (immutable tags, lifecycle policy) | M1 |
| Database | Amazon RDS for PostgreSQL 17, `db.t4g.micro`, TLS | M1 |
| Secrets and settings | AWS Secrets Manager, SSM Parameter Store | M1 |
| Compute | Amazon ECS on Fargate and Fargate Spot | M2 |
| Traffic | Application Load Balancer, path and header rules | M2, M3 |
| CI/CD | gitlab.com CI, Docker-in-Docker, AWS CLI | M4 |
| Observability | CloudWatch Logs (`awslogs`), CloudWatch dashboard | M5 |
| Scaling | Application Auto Scaling, target tracking | M5 |
| Safe releases | ECS blue/green deployments | M6 |
| Access control | IAM task execution role, task roles | M7 |

## Key engineering features

Your build must show these properties (how you achieve them is up to you,
using what the modules taught):

- **Disposable by design:** every session-only resource can be destroyed and
  re-created from code, in a known order, with no manual steps.
- **No secrets at rest in your code or state:** the DB password exists only
  in Secrets Manager, and the pipeline has no AWS key at all.
- **Everything tracked in Terraform:** no AWS resource is created by hand in
  the console or CLI.
- **Traceable releases:** every running image maps to one Git commit.
- **Safe to fail:** a bad release can't take the app down, and you can go
  back in seconds.
- **Least privilege, proven:** every identity (CI, execution role, task
  roles) can do only its job, and you can show a denied call.
- **Elastic on its own:** the backend grows and shrinks with load, without
  you touching it.

## Validation & testing

Run each check below and save the output in `docs/capstone/`. A correct
result is described for each.

| Check | What to run | Correct result |
|---|---|---|
| Database is private | `nc` from your laptop to the RDS host on `5432` | Fails / times out |
| No NAT Gateway | `describe-nat-gateways` count | `0` |
| Routing | `curl` `/` and `/api/tasks` on the ALB | `200` HTML and a JSON list |
| Routing failure path | Backend at 0 tasks, `curl` both paths | `/api/tasks` `503`, `/` `200` |
| Seed is re-runnable | Run the seed task twice | Exit `0` both times; second run `INSERT 0 0` |
| Zero-downtime rollout | Request loop during a frontend deploy | Only `200` in `rollout.log` |
| Bad release | Push a broken backend image | Deploy job fails; ECS rolls back; loop still `200` |
| Blue/green | Test header vs. normal route during a deploy | Test route = new version, normal = old, until the switch |
| Blue/green rollback | `stop-service-deployment --stop-type ROLLBACK` in bake time | Production rule back on the original target group |
| Scaling | 5-minute `ab` load test | Scaling activity 1 → 3 and back; mix of Spot and regular tasks |
| Least privilege | IAM probe with backend role, then frontend role | Backend: 2 allowed, 3 denied. Frontend: all denied |
| Teardown | Destroy in reverse order; list load balancers and NAT Gateways | Nothing left |

## Output & usage notes

- The ALB address changes every session, because the ALB is re-created.
  Share it only during a session.
- The database starts empty every session; only the 3 seed tasks are
  guaranteed. Anything you add is gone after `terraform destroy`.
- With only 1 backend task, you can't see the 70/30 Spot split — check it
  while the backend is scaled to 3.
- Plain HTTP only: there's no custom domain in this capstone, so no HTTPS
  certificate.

## Documentation

Keep your evidence in your repo under `docs/capstone/` as you build — not
at the end. This is what the **Documentation Gate** checks. Real output
only.

- `terraform apply` / `terraform destroy` output for `10-foundation`,
  `20-data` and `30-track-a`.
- The failed `nc` to RDS, and the NAT Gateway count.
- `curl` results for `/`, `/api/tasks`, and the backend-at-0 test.
- `rollout.log` summary (`sort | uniq -c`) from a zero-downtime rollout.
- The circuit-breaker rollback (`describe-services` deployments output).
- Blue/green: test vs. normal route output, `describe-rules` output, and
  the rollback status.
- IAM probe logs for both roles.
- Scaling activities, the Spot/regular task list, and a dashboard screenshot.
- Teardown proof: no load balancers, no NAT Gateways.
- A short explanation, in your own words, of your daily apply/destroy
  order and why it's in that order.

## Checkpoint (self-assessed)

- [ ] 1. VPC: 2 AZs, 2 public + 2 private subnets, no NAT Gateway.
- [ ] 2. Security-group chain with `/32` ingress; RDS only from `dcta-app-sg`.
- [ ] 3. Three stacks, S3 state with locking, exact provider versions, `terraform_remote_state`.
- [ ] 4. ECR: immutable tags, last 2 kept, `amd64` images tagged with Git SHAs.
- [ ] 5. Credentials only in Secrets Manager; no password in Git or state.
- [ ] 6. RDS private and re-creatable with no manual steps; `nc` from my laptop fails.
- [ ] 7. Task definitions: Fargate, `X86_64`, 256/512, SHA tags, DB settings via `secrets`.
- [ ] 8. 70/30 Spot strategy; the seed task exits `0` and is re-runnable.
- [ ] 9. `/api/*` → backend, default → frontend; backend-at-0 gives `503` on `/api/tasks`, `200` on `/`.
- [ ] 10. Merge to `main` builds, pushes and deploys with a scoped CI identity; the job waits for a stable service.
- [ ] 11. Zero-downtime frontend rollout; a broken image rolls back with no user errors.
- [ ] 12. Backend blue/green with a test route and 5-minute bake; I rolled one back.
- [ ] 13. Logs in Terraform-managed groups (≤ 7 days); dashboard shows CPU and ALB metrics.
- [ ] 14. Backend scaled 1 → 3 → 1 under load; I saw Spot and regular tasks.
- [ ] 15. Task roles scoped; the probe's denied calls are denied.
- [ ] 16. I can run the 15-minute defense and finish with a clean, verified teardown.

## Project scope

**This capstone does:**

- run one reference app on ECS Fargate in one AWS account and one region;
- automate build and deploy from gitlab.com;
- show safe releases, automatic scaling and least-privilege access;
- rebuild and tear down the whole environment every session.

**This capstone does not:**

- use a custom domain, Route 53 or HTTPS certificates;
- run in private subnets behind a NAT Gateway (containers use public IPs to
  reach ECR — a training shortcut, not a production pattern);
- keep data between sessions, take backups, or run across regions;
- include container image vulnerability scanning in the pipeline;
- cover application code changes beyond the three from M1.
