```
Project:      AWS Containers Bootcamp (Self-Paced, Hands-On)
Project Code: DCTA-AWS
Epic:         E003 — Track B: AWS EKS Path
Version:      v6.0
Date:         2026-09-28
Source PRD:   PRD: AWS Containers Bootcamp (Self-Paced, Hands-On), v1.0, 2026-09-28
Source Arch:  ARCH-DCTA-AWS-v6.0
```

## ADR-DCTA-AWS-E003-01: EKS control plane with a single EC2 Spot managed node

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 3 Tech Stack: "Amazon EKS (control plane + a single EC2 Spot managed node, `t3.small` / `t4g.small`)." PRD Epic 3 M9: "single EC2 Spot node... in public subnets, integrated into the Module 1 VPC." PRD Epic 3 AC 1: "EKS cluster running on a single EC2 Spot worker node with $0 NAT Gateway cost." PRD Epic 3 Out of Scope: "Multi-node or highly-available node groups (single Spot node only)."

### Rationale
- Satisfies Epic 3 AC 1 exactly.
- A single Spot node (rather than On-Demand, or more than one node) is the PRD's explicit cost-control mechanism for Track B, whose EKS control plane already carries its own hourly charge independent of node count — minimizing node count and using Spot pricing on the one node that exists directly targets the $5 ceiling (Epic 4 AC 1).
- Placing the node in the public subnets (ADR-DCTA-AWS-E001-02) is what lets it reach ECR without a NAT Gateway, satisfying the "$0 NAT Gateway cost" clause of AC 1 directly.

### Consequences
**Positive**: Satisfies AC 1 exactly; the EKS control plane's own per-hour charge is unavoidable regardless of node count, so keeping node count at exactly one avoids compounding that cost with EC2 charges from a second or third node.

**Negative**: A single node is a single point of failure for all of Track B's workloads and, being Spot capacity, can be reclaimed mid-lab; this is a named risk in the PRD itself (§6 Risks: "the single EC2 Spot node backing EKS could be reclaimed mid-lab") and its mitigation (ArgoCD self-heal, per M14, plus frequent `terraform apply`/Git-commit checkpoints) is addressed in ADR-DCTA-AWS-E003-07 and ARCH-DCTA-AWS-v6.0 §8.

### Affected Epics
Primary: E003 — Track B: AWS EKS Path
Also affects: E001 — Shared Infrastructure Foundation (shares the zero-NAT public-subnet design, ADR-DCTA-AWS-E001-02)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track B compute), §3.2 (Component table — EKS cluster + 1 EC2 Spot node)

---

## ADR-DCTA-AWS-E003-02: Node instance size — recommend `t3.medium`/`t4g.medium` over the PRD's `t3.small`/`t4g.small`

### Status
Proposed — pending program-sponsor approval (see ARCH-DCTA-AWS-v6.0 §9, Open Question 2)

### Type
Decision — alternatives were evaluated

### Context
PRD Epic 3 Tech Stack names `t3.small`/`t4g.small` as the node instance types. PRD Epic 3 Out of Scope confirms "single Spot node only" — no second node to spread load across. The full Track B capstone stack that must run on that one node includes: EKS system pods (`kube-proxy`, VPC CNI, CoreDNS), External Secrets Operator, ingress-nginx, ArgoCD, Prometheus Operator, Grafana, Kubernetes Metrics Server, and the frontend and backend application Deployments (Epic 3 M11–M15). Under the default Amazon VPC CNI, both `t3.small` and `t4g.small` are limited to **11 pods per node** and carry only 2 GiB of RAM. Counting only the pods explicitly named across M11–M15 already approaches or exceeds that pod ceiling before accounting for system pods, and 2 GiB is tight for Prometheus and Grafana running alongside everything else.

### Options Considered

#### Option 1: Keep `t3.small`/`t4g.small` as specified in the PRD
- **Pros**: No deviation from the PRD; matches the "low-cost" framing of Module 9's title exactly; lowest possible per-hour EC2 charge for the node.
- **Cons**: The default VPC CNI pod ceiling (11 pods) and 2 GiB RAM do not have enough headroom for the full M15 capstone stack (ESO, ingress-nginx, ArgoCD, Prometheus, Grafana, Metrics Server, 2 app Deployments, plus system pods) to schedule successfully; without a mitigation (e.g., CNI prefix delegation), learners following the syllabus module-by-module would hit `Insufficient pods` or OOM scheduling failures precisely at the observability module (M15) or the capstone integration (M16) — the two modules where a failure is most costly to a learner's schedule and confidence.

#f Option 2: Move to `t3.medium`/`t4g.medium` Spot
- **Pros**: Raises the default-CNI pod ceiling to 17 and RAM to 4 GiB, giving realistic headroom for the M15 stack; Spot pricing keeps the incremental hourly cost increase small relative to the node-count-of-one savings already locked in by ADR-DCTA-AWS-E003-01; keeps the "single node" constraint from Epic 3 Out of Scope fully intact — this changes size, not count.
- **Cons**: Deviates from the PRD's explicitly named instance types, requiring program-sponsor sign-off before content is authored (Open Question 2); modestly raises the per-live-hour cost of every Track B session (roughly $0.007/hr more at Spot pricing, `ap-southeast-1` planning rate), which must be re-checked against the $5 ceiling in the cost model.

### Decision
Recommend Option 2 (`t3.medium`/`t4g.medium` Spot), documented here as a proposed deviation pending explicit program-sponsor approval, with `t3.small`/`t4g.small` retained in the curriculum as a documented fallback only if the observability stack (M15) is trimmed to fit (fewer Grafana/Prometheus resource requests, or dropping one non-essential component).

### Rationale
Epic 3 AC 7 requires "HPA scales the backend under load, with scaling events visible in Grafana" and Epic 3 AC 6 requires ArgoCD auto-sync of "the Helm charts" (plural, implying frontend and backend both) — both acceptance criteria assume the full observability and GitOps stack is running simultaneously alongside the application. A node that cannot schedule that combination does not satisfy those acceptance criteria at all, which is a more severe risk to Epic 4 AC 1 (self-contained, i.e. it doesn't just risk going over budget, it risks the module not working) than the small additional per-hour cost of a larger Spot instance.

### Consequences
**Positive**: The full M15/M16 capstone stack has realistic headroom to schedule and run without a learner needing to debug pod-scheduling failures as their first exposure to resource limits; keeps the "single node" cost-control principle from Epic 3 Out of Scope fully intact.

**Negative**: Requires an explicit amendment to the PRD's named instance types before Track B content can be authored with confidence (blocking Open Question 2); the cost model in ARCH-DCTA-AWS-v6.0 §8 must be recomputed with the larger instance size, and Track B's live-hour budget (already the tighter of the two tracks) tightens further.

### Affected Epics
Primary: E003 — Track B: AWS EKS Path
Also affects: E004 — Program Operations, FinOps Guardrails & Capstone Defense (changes the Track B cost-per-live-hour figure the FinOps guardrails are built around)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track B node instance size), §8 (NFR Fulfillment — Budget ceiling), §9 Open Question 2

---

## ADR-DCTA-AWS-E003-03: Terraform owns EKS cluster lifecycle; `eksctl` is a diagnostic utility only

### Status
Accepted

### Type
Decision — alternatives were evaluated

### Context
PRD Epic 3 Tech Stack lists both `eksctl` and the `terraform-aws-modules/eks/aws` module as required tooling. PRD Epic 3 M9 builds the cluster with the Terraform EKS module. PRD Epic 3's final acceptance criterion (AC 8) and M16's closing step both name `eksctl delete cluster` as part of the capstone teardown, alongside `terraform destroy`. `eksctl` is designed to manage (and delete) clusters that it itself created via CloudFormation stacks; pointed at a cluster built by Terraform, `eksctl delete cluster` has no CloudFormation stack to operate against and is not a supported teardown path for that cluster.

### Options Considered

#### Option 1: Follow the PRD literally — build with Terraform (M9), delete with `eksctl delete cluster` (M16)
- **Pros**: Matches the PRD's tool list and M16's literal teardown step without modification.
- **Cons**: Does not work as described — `eksctl` cannot delete a cluster it did not create, so M16's scripted teardown would fail at the worst possible moment (immediately after the live capstone defense, per Epic 3 AC 8), directly threatening Epic 4 AC 1's zero-tolerance budget ceiling if the cluster is left running because the documented teardown command errors out.

#### Option 2: Terraform owns the full cluster lifecycle (create in M9, destroy in M16); `eksctl` is retained only for read/diagnostic commands (e.g., `eksctl get cluster`, `eksctl utils describe-stacks` for inspection) that do not touch lifecycle
- **Pros**: `terraform destroy` reliably tears down everything Terraform created, including the cluster, matching how every other resource in this design is torn down (ADR-DCTA-AWS-E004-01); `eksctl` remains present in the curriculum and toolchain for its genuinely useful inspection/diagnostic commands, so the PRD's tool list is not simply dropped, only its lifecycle role is corrected.
- **Cons**: Deviates from a literal reading of M16's teardown script, requiring a curriculum content fix before Track B modules are authored (Open Question 3); learners must understand *why* `eksctl delete cluster` is not used here, which is itself additional (if valuable) content about tool ownership.

### Decision
Option 2. `terraform destroy` (against the `40-track-b-cluster` stack) is the sole cluster-lifecycle teardown command taught for M16; `eksctl` is retained in the toolchain for diagnostics only.

### Rationale
Epic 4 AC 1 is an absolute, zero-tolerance budget ceiling ("no learner's cumulative AWS spend... exceeds $5.00 USD at any point"). A teardown command that predictably fails against a Terraform-built cluster is a direct risk to that ceiling, occurring at the single highest-stakes moment in the program (immediately after the live defense, when a learner's attention has just shifted from the exercise to relief that it's over). Correctness of the documented teardown path takes precedence over matching the PRD's literal tool list for this one step.

### Consequences
**Positive**: The M16 teardown reliably removes every billable EKS resource in one command, consistent with every other stack in this design; avoids a predictable, high-stakes failure mode at the worst point in the bootcamp schedule.

**Negative**: Requires flagging and correcting the PRD's M16 script before Track B content is authored (Open Question 3); slightly narrows `eksctl`'s taught role relative to what the PRD's tool list might imply to a curriculum author reading it without this ADR.

### Affected Epics
Primary: E003 — Track B: AWS EKS Path
Also affects: E004 — Program Operations, FinOps Guardrails & Capstone Defense (M16's teardown is part of the Daily/Capstone FinOps SOP this epic owns)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track B cluster lifecycle tooling), §9 Open Question 3

---

## ADR-DCTA-AWS-E003-04: OIDC Federation and IAM Roles for Service Accounts (IRSA)

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 3 M10: "EKS OIDC provider, IAM Role for a Kubernetes Service Account with Secrets Manager read access." PRD Epic 3 AC 2: "A test Pod successfully assumes an IAM Role via IRSA/OIDC." PRD §2: "OIDC Federation & IAM Roles for Service Accounts / IRSA (Track B)."

### Rationale
Directly satisfies Epic 3 AC 2. IRSA is the standard EKS mechanism for letting a specific ServiceAccount (and therefore the Pods that use it) assume a scoped IAM role without static, long-lived AWS credentials stored in the cluster — the Kubernetes-native equivalent of the ECS Task Role split taught in Track A (ADR-DCTA-AWS-E002-05), and the prerequisite for External Secrets Operator's own AWS access in Module 11.

### Consequences
**Positive**: Satisfies AC 2 exactly; establishes credential-free Pod-to-AWS identity that Module 11's External Secrets Operator depends on directly, so this module's output is a hard input to the next.

**Negative**: The IRSA trust policy must be scoped to the exact ServiceAccount name and namespace ESO will use in Module 11 — building M10 against a throwaway test ServiceAccount instead of ESO's actual one (a hidden dependency not stated in the syllabus) forces a rebuild of the trust policy in M11, wasting a learner's time and live-hours against the Epic 4 budget ceiling if not caught.

### Affected Epics
Primary: E003 — Track B: AWS EKS Path
Also affects: None

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track B identity), §6 (Integration and API Design — ESO → Secrets Manager row)

---

## ADR-DCTA-AWS-E003-05: External Secrets Operator syncing the shared Secrets Manager secret

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 3 M11: "`ClusterSecretStore` + `ExternalSecret` syncing RDS credentials into a native Kubernetes `Secret`." PRD Epic 3 AC 3: "RDS credentials appear as a native Kubernetes `Secret`, synced by ESO with no credentials committed to Git."

### Rationale
Directly satisfies Epic 3 AC 3. Reusing the single Secrets Manager secret created in Epic 1 (ADR-DCTA-AWS-E001-05) rather than creating a second, Track-B-specific secret avoids doubling the per-secret Secrets Manager charge for the same underlying DB credentials, and reinforces that both tracks are deploying the same application against the same database.

### Consequences
**Positive**: Satisfies AC 3 exactly, including its explicit "no credentials committed to Git" requirement, since the Kubernetes `Secret` is generated at sync time from AWS, never authored by hand; reuses Epic 1's existing secret at no additional Secrets Manager cost.

**Negative**: Couples Track B's secret availability to Epic 1's foundation stack and to the IRSA trust policy built in M10 being scoped to ESO's actual ServiceAccount (ADR-DCTA-AWS-E003-04); if RDS is ephemeral (ADR-DCTA-AWS-E001-04) and its endpoint or credentials rotate on recreation, the `ExternalSecret`'s refresh interval determines how quickly the cluster picks up the change, which must be tuned short enough not to strand a session on stale credentials.

### Affected Epics
Primary: E003 — Track B: AWS EKS Path
Also affects: E001 — Shared Infrastructure Foundation (reuses the Epic 1 secret, ADR-DCTA-AWS-E001-05)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track B secrets sync), §6 (Integration and API Design — sequence diagram)

---

## ADR-DCTA-AWS-E003-06: NGINX Ingress exposed via its own Service-provisioned Load Balancer, not an AWS Load Balancer Controller/ALB Ingress

### Status
Accepted

### Type
Decision — alternatives were evaluated

### Context
PRD Epic 3 M13: "deploy NGINX Ingress via Helm; frontend `Ingress` resource; verify AWS auto-provisions a Load Balancer." PRD Epic 3 AC 5: "NGINX Ingress exposes the frontend via an AWS-provisioned Load Balancer URL (no Route 53)." The PRD names NGINX Ingress Controller specifically but does not specify the mechanism by which AWS provisions the Load Balancer behind it — there are two common patterns on EKS.

### Options Considered

#### Option 1: AWS Load Balancer Controller + ALB-backed `Ingress` objects
- **Pros**: More AWS-native integration (ALB target-group-per-Pod, WAF/ACM integration available); closer to a "real" production EKS ingress pattern some teams use.
- **Cons**: Requires installing and IRSA-authorizing a second controller beyond ingress-nginx itself, adding its own IAM role, Helm release, and reconciliation loop; no PRD acceptance criterion asks for ALB-specific features (WAF, ACM, path-based ALB rules) — everything AC 5 requires (an AWS-provisioned LB URL, no Route 53) is satisfied without it; the extra controller is pure incremental cost and complexity against Epic 4's budget ceiling with no corresponding requirement.

#### Option 2: ingress-nginx's own `Service` of type `LoadBalancer`, which the in-cluster AWS cloud-provider integration turns into a Network Load Balancer automatically
- **Pros**: One fewer controller to install, IRSA-authorize, and maintain; the Load Balancer that results is exactly what AC 5 asks for (an AWS-provisioned URL) with no additional AWS-specific Kubernetes controller; this is also the pattern already assumed in Module 15, where Grafana is exposed via an Ingress *path* on the same Load Balancer rather than a Load Balancer of its own — consistent with the single-LB cost-avoidance pattern used throughout Track B.
- **Cons**: Fewer AWS-native integration points (no native WAF/ACM attachment) if the curriculum ever wanted to extend into those areas — not required by any current acceptance criterion, but a constraint on future extension.

### Decision
Option 2: ingress-nginx installed via Helm, exposed through its own `Service` of type `LoadBalancer`, which provisions one Network Load Balancer for the entire Track B capstone (frontend, backend API, and Grafana, all routed by Ingress path/host rules behind that one LB).

### Rationale
Epic 3 AC 5 only requires an AWS-provisioned Load Balancer URL exposing the frontend, with no Route 53 involvement — a requirement Option 2 satisfies completely without adding a second controller. Epic 4 AC 1's budget ceiling weighs against introducing an unrequired component (the AWS Load Balancer Controller) whose only benefit is ALB-specific features no acceptance criterion asks for.

### Consequences
**Positive**: One fewer Helm release, one fewer IRSA role, and one fewer moving part for a learner to debug; the resulting single Load Balancer is reused across Modules 13 and 15 (frontend/backend, then Grafana), directly supporting the single-LB cost-avoidance pattern that keeps Track B's traffic-plane cost to one Load Balancer for the whole capstone.

**Negative**: If a future revision of this program wants ALB-specific features (native WAF attachment, per-path ALB target groups, ACM-issued TLS on the LB itself), this decision would need to be revisited and the AWS Load Balancer Controller introduced at that point.

### Affected Epics
Primary: E003 — Track B: AWS EKS Path
Also affects: None (Grafana's shared use of this Load Balancer is addressed in ADR-DCTA-AWS-E003-08)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track B ingress and load balancer provisioning), §7.2 (Track B capstone deployment diagram)

---

## ADR-DCTA-AWS-E003-07: ArgoCD GitOps, auto-sync/prune/self-heal, reading from the shared GitLab repo

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 3 M14: "deploy ArgoCD; connect to GitLab; ArgoCD `Application` auto-syncing the Helm charts." PRD Epic 3 AC 6: "ArgoCD auto-syncs the Helm charts from GitLab on a Git change."

### Rationale
Directly satisfies Epic 3 AC 6. Self-heal is enabled specifically because it is the design's stated mitigation for the PRD's own named risk — Spot node reclamation (Epic 3 M9, §6 Risks) — automatically reapplying the desired state (including re-scheduling workloads) once a replacement node joins the cluster after a Spot interruption, without requiring the learner to manually redeploy.

### Consequences
**Positive**: Satisfies AC 6 exactly; self-heal directly mitigates the PRD's own named Spot-interruption risk, turning ArgoCD from a pure deployment tool into a resilience mechanism the curriculum can point to as intentional, not accidental.

**Negative**: ArgoCD requires a read-only GitLab deploy token if the repository is private (a dependency not stated explicitly in the syllabus but required for M14 to function); self-heal can also mask a learner's manual `kubectl` experimentation by silently reverting it to the Git-declared state, which is worth calling out explicitly in the guided activity so it doesn't read as unexplained behavior.

### Affected Epics
Primary: E003 — Track B: AWS EKS Path
Also affects: None

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track B delivery), §6 (Integration and API Design — sequence diagram), §8 (NFR Fulfillment — Resilience, Spot interruption)

---

## ADR-DCTA-AWS-E003-08: Prometheus Operator, Grafana, Metrics Server, and HPA — sized to fit the single-node ceiling

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 3 M15: "Prometheus/Grafana via ArgoCD, Grafana exposed via Ingress, backend `HorizontalPodAutoscaler` at 70% CPU, load test capturing HPA scaling events." PRD Epic 3 AC 7: "HPA scales the backend under load, with scaling events visible in Grafana."

### Rationale
Directly satisfies Epic 3 AC 7. Grafana is exposed via an Ingress *path* on the single existing Load Balancer (ADR-DCTA-AWS-E003-06) rather than its own Load Balancer, which the PRD does not require and which would add a second LB's hourly charge for no acceptance-criteria benefit. All four components (Prometheus Operator, Grafana, Metrics Server, HPA) are deployed via ArgoCD (ADR-DCTA-AWS-E003-07), consistent with the design's Terraform-installs-the-platform / ArgoCD-owns-the-workloads ownership split.

### Consequences
**Positive**: Satisfies AC 7 exactly; sharing the Load Balancer with the application traffic avoids a second LB's cost; the ArgoCD-managed installation means the observability stack self-heals from the same Spot-interruption risk as the application workloads (ADR-DCTA-AWS-E003-07).

**Negative**: Prometheus and Grafana together are among the largest memory consumers this design places on the single Spot node (alongside ArgoCD itself), which is the direct motivation for ADR-DCTA-AWS-E003-02's node-sizing recommendation; if that ADR's `t3.medium`/`t4g.medium` recommendation is not approved, this module's resource requests must be trimmed (reduced Prometheus retention/scrape scope, Grafana resource limits lowered) to fit the smaller node, which is documented as the fallback path in ADR-DCTA-AWS-E003-02.

### Affected Epics
Primary: E003 — Track B: AWS EKS Path
Also affects: None (node-capacity risk is owned by ADR-DCTA-AWS-E003-02)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track B observability and scaling), §8 (NFR Fulfillment — Scalability, Observability)
