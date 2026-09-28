# Capstone — Track B: Task Manager on EKS with GitOps

In this capstone you bring together M1 and M9–M15: you'll run the
task-manager app on **Amazon EKS**, with secrets synced by the **External
Secrets Operator**, traffic through **one NGINX Ingress load balancer**,
everything deployed from Git by **Argo CD**, watched with **Prometheus and
Grafana**, and scaled by a **Horizontal Pod Autoscaler** — all built with
**Terraform** and rebuilt from scratch every session.

> **Disclaimer**
>
> The team and scenario below are fictional. This capstone was written for
> the DevOps Bootcamp's *Containerization in AWS* phase and uses the same
> reference task-manager app (React, Node.js/Express, PostgreSQL) you
> deployed to Minikube in the local phase.

## Getting started

### Prerequisites

You need everything from M1 and M9–M15 working:

- Your AWS sandbox account (region `ap-southeast-1`) and your Terraform
  state bucket name.
- Your gitlab.com project with the M1 app changes, the `task-app` chart in
  `deploy/charts/task-app/`, and a read-only GitLab deploy token.
- Tools: Terraform 1.11+, AWS CLI v2, Docker with `buildx`, `kubectl`,
  `helm` 3, `eksctl` (read-only), `git`, `curl`, `jq`, `dig`, `ab`.
- The daily routine: settings → `10-foundation` → `20-data` →
  `40-track-b-cluster` → `41-track-b-platform` → `kubectl apply -f
  deploy/argocd/` at the start; destroy in reverse order at the end.

### Data setup

The database is empty every session. The data comes from the reference
app's schema and sample rows (see **Data sources**) — your chart's seed Job
loads them every time Argo CD syncs the app. There's nothing to download.

## Business problem

> Can a small team run its task-manager app on Kubernetes in AWS so that
> Git is the only way to change what's running, no password ever lives in
> Git, the app scales by itself, and the whole cluster can be rebuilt from
> the repository in minutes?

This capstone helps the (fictional) TaskFlow team:

- deploy by committing to Git — and undo by reverting;
- keep the database password in one place (AWS), never in the repo;
- recover automatically when a node is replaced or someone edits the
  cluster by hand;
- see what the app is doing, and let it grow under load.

## Requirements / acceptance criteria

Each requirement comes from one module's Lab exercise.

**Foundation (M1)**

1. A VPC in `ap-southeast-1` across 2 AZs with **2 public and 2 private
   subnets**, an Internet Gateway, **no NAT Gateway**, and public subnets
   that give instances a public IP.
2. Security groups `dcta-alb-sg` / `dcta-app-sg` / `dcta-rds-sg`: internet
   traffic only from your `/32`; RDS allows `5432` **only** from
   `dcta-app-sg`.
3. Terraform split into `10-foundation`, `20-data`, `40-track-b-cluster`
   and `41-track-b-platform`, with remote S3 state (`use_lockfile = true`),
   exact provider/module/chart versions, and `terraform_remote_state`
   between stacks.
4. ECR repositories `dcta-frontend` and `dcta-backend` with immutable tags,
   keeping the last 2 images; images are `linux/amd64` and tagged with a
   Git SHA.
5. The DB credentials are in Secrets Manager; the DB host and name are in
   Parameter Store. **No password in Git or in any Terraform state file.**
6. RDS PostgreSQL `db.t4g.micro` in the private subnets, not publicly
   accessible, re-creatable every session with no manual steps.
   **Failure proof:** a connection from your laptop to RDS on `5432` fails.

**Cluster and identity (M9–M10)**

7. EKS cluster `dcta-eks`, Kubernetes **1.36**, built with
   `terraform-aws-modules/eks/aws` **21.26.0**; API endpoint reachable only
   from your `/32`.
8. Exactly **one** Spot node (`t3.medium` / `t3a.medium`) in a public
   subnet, with `dcta-app-sg` attached and prefix delegation on (pod
   capacity `110`); a pod can reach RDS.
9. IAM role `dcta-eso-irsa` usable **only** by
   `system:serviceaccount:external-secrets:external-secrets`, allowed only
   to read the DB secret and the two DB parameters. **Failure proof:** a pod
   with another ServiceAccount can't read the DB secret, and the allowed
   pod can't read another secret.

**Secrets and app (M11–M12)**

10. ESO installed by Terraform (`helm_release`) using the IRSA role.
11. Two `ClusterSecretStore`s and two `ExternalSecret`s create
    `db-credentials` (`DB_USER`, `DB_PASSWORD`) and `db-config` (`DB_HOST`,
    `DB_NAME`) in `task-app`. **Failure proof:** an `ExternalSecret` pointing
    at a forbidden secret reports access denied without overwriting the
    existing `Secret`.
12. One `task-app` chart with `values-eks.yaml`: ECR SHA-tagged images, no
    in-cluster database, DB settings through the ESO Secrets, requests,
    memory limits, `/health` probes, and a re-runnable seed Job.
    **Failure proof:** a bad image tag is diagnosed from pod events while
    the old pod keeps serving.

**Traffic and delivery (M13–M14)**

13. F5 NGINX Ingress Controller installed by Terraform: one NLB, only your
    `/32` allowed, Ingresses without a host allowed, and `wait = true`.
14. The chart's `Ingress` sends `/api` to the backend and `/` to the
    frontend. **Failure proof:** with the backend at 0, `/api/tasks` returns
    a 5xx while `/` still works.
15. Argo CD installed by Terraform, UI private; the deploy token is created
    from your shell and never in Git.
16. Argo CD `Application`s `platform`, `task-app-secrets` and `task-app`
    with automated sync, prune and self-heal; a commit alone changes the
    app. **Failure proof:** a deleted Deployment is restored by self-heal,
    and a bad commit goes `Degraded` without taking the app down, then
    recovers with `git revert`.

**Operations (M15)**

17. metrics-server and `kube-prometheus-stack` installed through Argo CD,
    sized for one node; Grafana's admin password created from your shell;
    Grafana reachable at `grafana.dcta.test` on the same load balancer.
18. An HPA scales the backend **1 → 3 pods at 70% CPU** under a 5-minute
    load test and back again, visible in Grafana. **Failure proof:** without
    a CPU request, the HPA shows `<unknown>` and says why.

**Defense and teardown (M16)**

19. A live, 15-minute defense following the rubric's agenda, including a
    **decision matrix** of at least 5 technical choices you made (see
    **Deliverable**).
20. Teardown with `terraform destroy` in reverse order —
    `41-track-b-platform`, then `40-track-b-cluster`, then `20-data` — and
    proof that no load balancer or NAT Gateway is left.

## Deliverable

This is a running system plus the evidence that it works, and one
document you write: the **decision matrix**.

What the facilitator will look at:

- **Your app** at the NLB address: the task-manager UI on `/`, JSON on
  `/api/tasks`.
- **Argo CD** (through port-forward): every Application `Synced` and
  `Healthy`.
- **Grafana** at `grafana.dcta.test`: the HPA replica graph during a load
  test.
- **Your repository:** `app/`, `infra/<stack>/`, `deploy/`, and
  `docs/capstone/` with your evidence (see **Documentation**).

### Decision matrix

Save it as `docs/capstone/decision-matrix.md`. At least **5 rows**, each
about a real choice in *your* build:

| Decision | What I chose | Alternative I considered | Trade-off | Evidence from my build |
|---|---|---|---|---|

Pick from things like: how pods get AWS credentials, how the DB password
reaches the cluster, how traffic enters the cluster, how deployments
happen, node size and pod capacity, HPA settings, teardown order. Be ready
to defend each row in the Q&A.

## Data sources

| Source | Where it comes from | Type |
|---|---|---|
| Task schema and 3 sample tasks | The reference app's `app/database/init.sql`, turned into the re-runnable `seed.sql` in M1 and copied into the chart | Static, small |
| Tasks you create in the demo | Typed in through the app | Live, in RDS for one session |
| Failure-path inputs | A forbidden secret, a bad image tag, a deleted Deployment, a missing CPU request | Deliberate, made up for tests |

## Project architecture

```text
Developer ──git push──► gitlab.com repo (deploy/argocd, deploy/platform, deploy/apps, deploy/charts)
                                   ▲
                                   │ Argo CD pulls (deploy token)
EKS cluster (1 Spot node) ─────────┘
 ├─ external-secrets ──IRSA──► Secrets Manager + Parameter Store ──► Secrets db-credentials, db-config
 ├─ nginx-ingress ◄── NLB ◄── browser (your /32)
 │     ├─ /api ─► backend pods (HPA 1–3) ──TLS :5432──► RDS (private subnets)
 │     ├─ /    ─► frontend pod
 │     └─ grafana.dcta.test ─► Grafana
 ├─ argocd (self-heal, prune)
 └─ monitoring: Prometheus, Grafana, kube-state-metrics, node-exporter;  kube-system: metrics-server ─► HPA
```

What drives it:

- **You**, each session: `terraform apply` on `10-foundation` → `20-data`
  → `40-track-b-cluster` → `41-track-b-platform`, then `kubectl apply -f
  deploy/argocd/`; `terraform destroy` in reverse at the end.
- **Argo CD**, continuously: pulls Git and keeps the cluster matching it.
- **ESO, the HPA, and Kubernetes itself**, continuously: sync secrets,
  scale pods, replace failed ones.

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
| Infrastructure as code | Terraform 1.11+, AWS provider 6.66.0, Helm provider 3.3.0, S3 state with native locking | M1, M9, M11 |
| Network, registry, database, secrets | VPC, ECR, RDS PostgreSQL 17, Secrets Manager, Parameter Store | M1 |
| Kubernetes | Amazon EKS 1.36 via `terraform-aws-modules/eks/aws` 21.26.0, 1 Spot node, VPC CNI prefix delegation | M9 |
| Workload identity | OIDC + IAM Roles for Service Accounts | M10 |
| Secrets sync | External Secrets Operator 2.11.0 | M11 |
| Packaging | Helm chart `task-app` with `values-eks.yaml` | M12 |
| Traffic | F5 NGINX Ingress Controller 2.7.3 + one NLB | M13 |
| GitOps | Argo CD 10.9.2 | M14 |
| Observability and scaling | kube-prometheus-stack 91.8.1, metrics-server 3.14.0, HPA `autoscaling/v2` | M15 |

## Key engineering features

Your build must show these properties (how you achieve them is up to you,
using what the modules taught):

- **Git is the only way in:** after bootstrap, every change to the app
  reaches the cluster through a commit.
- **Rebuildable from the repo:** a brand-new cluster reaches `Synced` and
  `Healthy` with one `kubectl apply` of your `Application`s.
- **No secrets at rest in the repo or state:** the DB password lives only
  in Secrets Manager; the deploy token and Grafana password live only in
  the cluster.
- **Workload identity, least privilege, proven:** one role for one
  ServiceAccount, with a denied-call test.
- **Self-healing:** manual drift and deleted objects are put back.
- **Elastic and observable:** the backend scales on CPU, and you can show
  it in Grafana.

## Validation & testing

Run each check below and save the output in `docs/capstone/`. A correct
result is described for each.

| Check | What to run | Correct result |
|---|---|---|
| Database is private | `nc` from your laptop to the RDS host on `5432` | Fails / times out |
| No NAT Gateway | `describe-nat-gateways` count | `0` |
| One Spot node, prefix delegation | `kubectl get nodes` with labels; pod capacity | 1 node, `SPOT`, capacity `110` |
| Pod → RDS | `pg_isready` from a pod | `accepting connections` |
| IRSA | Test pod with the ESO ServiceAccount; then with `default` | Allowed pod reads the DB secret name only; `default` can't |
| ESO | `get clustersecretstores`, `get externalsecrets` | Stores ready; both `SecretSynced` |
| ESO failure path | Point an `ExternalSecret` at a forbidden secret | Access-denied status; existing `Secret` unchanged |
| Routing | `curl` `/` and `/api/tasks` on the NLB | `200` HTML and a JSON list |
| Routing failure path | Backend at 0 (before Argo CD) | `/api/tasks` 5xx, `/` `200` |
| GitOps | Push a visible change | `OutOfSync` → `Synced`, change live |
| Self-heal | Delete the backend Deployment | Recreated by Argo CD |
| Bad commit | Push a non-existent image tag, then revert | `Degraded`, app still serving, then `Healthy` |
| Scaling | 5-minute `ab` load test | HPA 1 → 3 → 1; Grafana graph shows both steps |
| HPA failure path | Remove the CPU request in Git | HPA target `<unknown>` with a "missing request for cpu" event |
| Teardown | Destroy in reverse order; list load balancers and NAT Gateways | Nothing left |

## Output & usage notes

- The NLB address changes every session; so does its IP for
  `grafana.dcta.test`. Update your hosts file each session and remove the
  line afterwards.
- The database starts empty every session; only the 3 seed tasks are
  guaranteed.
- While self-heal is on, manual `kubectl` changes to Argo CD-managed
  objects are reverted — make changes in Git.
- Plain HTTP only: no custom domain, no HTTPS certificate.

## Documentation

Keep your evidence in your repo under `docs/capstone/` as you build — not
at the end. This is what the **Documentation Gate** checks. Real output
only.

- `terraform apply` / `terraform destroy` output for `10-foundation`,
  `20-data`, `40-track-b-cluster` and `41-track-b-platform`.
- The failed `nc` to RDS, the NAT Gateway count, and the node's pod
  capacity.
- IRSA test-pod logs: allowed pod and `default` pod.
- ESO: store and `ExternalSecret` status, the forbidden-secret status, and
  a `git grep` showing no password.
- `curl` results for `/`, `/api/tasks`, and the backend-at-0 test.
- Argo CD Application status before/after a commit; self-heal; bad commit
  and revert.
- HPA events, and a Grafana screenshot of the replica graph during the
  load test; the missing-CPU-request event.
- Teardown proof: no load balancers, no NAT Gateways.
- `decision-matrix.md`.
- A short explanation, in your own words, of your daily apply/destroy
  order and why it's in that order.

## Checkpoint (self-assessed)

- [ ] 1. VPC: 2 AZs, 2 public + 2 private subnets, no NAT Gateway, public IPs on launch.
- [ ] 2. Security-group chain with `/32` ingress; RDS only from `dcta-app-sg`.
- [ ] 3. Four stacks, S3 state with locking, exact versions, `terraform_remote_state`.
- [ ] 4. ECR: immutable tags, last 2 kept, `amd64` images tagged with Git SHAs.
- [ ] 5. Credentials only in Secrets Manager; no password in Git or state.
- [ ] 6. RDS private and re-creatable; `nc` from my laptop fails.
- [ ] 7. EKS 1.36 via module 21.26.0; API reachable only from my `/32`.
- [ ] 8. One Spot node with `dcta-app-sg`, pod capacity 110; a pod reaches RDS.
- [ ] 9. `dcta-eso-irsa` pinned to one ServiceAccount and scoped; denied tests pass.
- [ ] 10. ESO installed by Terraform with the IRSA role.
- [ ] 11. Both `ExternalSecret`s synced; forbidden-secret test reports access denied.
- [ ] 12. One chart + `values-eks.yaml` with SHA images, no in-cluster DB, ESO Secrets, requests/limits/probes, re-runnable seed; bad tag diagnosed from events.
- [ ] 13. NGINX Ingress: one NLB, `/32` only, host-less allowed, `wait = true`.
- [ ] 14. `/api` → backend, `/` → frontend; backend-at-0 gives a 5xx on `/api/tasks`.
- [ ] 15. Argo CD private; deploy token never in Git.
- [ ] 16. Three Applications with auto sync, prune, self-heal; commit, self-heal and revert all proven.
- [ ] 17. metrics-server and monitoring via Argo CD; Grafana at `grafana.dcta.test` with a shell-created password.
- [ ] 18. HPA 1 → 3 → 1 under load, visible in Grafana; missing-CPU-request failure shown.
- [ ] 19. Decision matrix with 5+ rows, ready to defend.
- [ ] 20. Teardown in reverse order; no load balancer or NAT Gateway left.

## Project scope

**This capstone does:**

- run one reference app on one EKS cluster with one node, in one AWS
  account and one region;
- deploy everything in the cluster from Git with Argo CD;
- sync secrets from AWS, expose the app through one load balancer, and
  scale the backend automatically;
- rebuild and tear down the whole environment every session.

**This capstone does not:**

- use a custom domain, Route 53 or HTTPS certificates;
- run more than one node, scale nodes, or run nodes in private subnets
  behind a NAT Gateway;
- keep data between sessions, take backups, or run across regions;
- set up alerting (Alertmanager is off) or log aggregation;
- cover application code changes beyond the three from M1.
