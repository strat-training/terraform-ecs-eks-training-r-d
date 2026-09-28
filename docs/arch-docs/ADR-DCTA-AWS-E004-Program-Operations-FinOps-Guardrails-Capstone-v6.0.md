```
Project:      AWS Containers Bootcamp (Self-Paced, Hands-On)
Project Code: DCTA-AWS
Epic:         E004 — Program Operations, FinOps Guardrails & Capstone Defense
Version:      v6.0
Date:         2026-09-28
Source PRD:   PRD: AWS Containers Bootcamp (Self-Paced, Hands-On), v1.0, 2026-09-28
Source Arch:  ARCH-DCTA-AWS-v6.0
```

## ADR-DCTA-AWS-E004-01: Split Terraform into independent-lifecycle stacks (persistent foundation vs. ephemeral data/compute)

### Status
Accepted

### Type
Decision — alternatives were evaluated

### Context
PRD Epic 4 names the "Daily FinOps SOP: `terraform apply`/`eksctl create` at the start of a lab session, `terraform destroy`/`eksctl delete` at the end of every session, enforced across all 16 modules" as a required feature, and Epic 4 AC 1 sets an absolute ceiling: "No learner's cumulative AWS spend in `ap-southeast-1` exceeds $5.00 USD at any point in the bootcamp." Epic 1's VPC, ECR repositories, and Secrets Manager secret carry effectively no idle cost; Epic 1's RDS instance, and every Track A/Track B compute, traffic, and identity resource, do carry an hourly cost while they exist (ARCH-DCTA-AWS-v6.0 §3.1, §8).

### Options Considered

#### Option 1: One monolithic Terraform stack for the whole bootcamp, destroyed and recreated in full every session
- **Pros**: Simplest mental model — one `terraform apply`, one `terraform destroy`, matching the PRD's SOP description most literally; no cross-stack output/input contracts to design or teach.
- **Cons**: Forces a full VPC, subnet, and security-group rebuild every single day across all 16 modules, even though those resources cost nothing while idle and gain nothing from being destroyed; needlessly increases both daily session-start latency and the daily failure surface (a VPC/subnet recreate is more likely to hit an AWS API rate limit or an unrelated transient error than a narrower compute-only apply); actively works against Epic 4 AC 1 by spending session time on rebuilding free resources instead of leaving budget-relevant time for the module's actual lab content.

#### Option 2: Independent-lifecycle stacks — a persistent foundation stack (VPC, ECR, Secrets Manager) built once, plus ephemeral stacks (RDS; Track A compute; Track B cluster; Track B platform) created and destroyed every session
- **Pros**: Directly matches cost to lifecycle — only the resources that actually bill by the hour are part of the daily create/destroy loop, which is what Epic 4 AC 1's budget ceiling actually depends on; daily session start is faster since the VPC/ECR/secret already exist; the four separate ephemeral stacks (data, Track A, Track B cluster, Track B platform) can be destroyed in a specific, safe order (addressed in ADR-DCTA-AWS-E004-03) rather than as one undifferentiated blob.
- **Cons**: Requires teaching `terraform_remote_state` output/input contracts between stacks (e.g., the foundation stack's VPC ID and app-tier security-group ID must be consumed by the data and compute stacks); a learner who runs stacks out of order, or who deletes the foundation stack by mistake, breaks every ephemeral stack that depends on it — a failure mode Option 1 does not have, since there is only one stack to run.

### Decision
Option 2: a persistent `10-foundation` stack (VPC, ECR, Secrets Manager, SSM parameters), plus ephemeral stacks — `20-data` (RDS), `30-track-a` (ECS/ALB/CloudWatch/CodeDeploy/IAM), `40-track-b-cluster` (EKS/OIDC/IRSA), `41-track-b-platform` (ESO/ingress-nginx/ArgoCD) — each destroyed and recreated per the daily SOP, while `10-foundation` persists for the full 2 weeks.

### Rationale
Epic 4 AC 1 is a zero-tolerance ceiling across the entire bootcamp, not a per-session target, so the daily create/destroy loop must be as fast and as narrowly scoped to actually-billing resources as possible — every minute spent rebuilding a free VPC is a minute a learner could otherwise spend on the module's content, and every unnecessary daily `apply`/`destroy` cycle on a resource is an unnecessary chance for a transient AWS error to cost a learner time (not directly AWS spend, but a real risk to the PRD's completion-rate goal, §1 Goal 1). Splitting by lifecycle is the direct architectural translation of the PRD's own Epic 4 feature description, which distinguishes "the start of a lab session" resources from resources implied to already exist.

### Consequences
**Positive**: Daily session start is fast (no VPC rebuild); the daily destroy operation only ever touches resources that actually bill, directly serving the $5 ceiling; the stack boundaries map cleanly onto the module-to-plane architecture (ARCH-DCTA-AWS-v6.0 §3.1), so learners can reason about "what exists right now" in terms of which stacks have been applied.

**Negative**: Introduces explicit `terraform_remote_state` contracts between stacks that must be documented and kept stable (a breaking output rename in `10-foundation` would break every ephemeral stack); a learner who accidentally destroys the foundation stack mid-bootcamp (rather than an ephemeral one) loses persistent state (ECR images, the ARN references other stacks depend on) that Option 1 would not have made possible to lose selectively.

### Affected Epics
Primary: E004 — Program Operations, FinOps Guardrails & Capstone Defense
Also affects: E001 — Shared Infrastructure Foundation (owns the persistent stack); E002 — Track A: AWS ECS & Fargate Path; E003 — Track B: AWS EKS Path (both consume outputs from the foundation and data stacks)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — State backend and stack topology), §7.1 (Environments — Terraform stack table and diagram), §3.1 (module-to-plane translation)

---

## ADR-DCTA-AWS-E004-02: Trivy as the mandatory container-image vulnerability scan gate in CI

### Status
Deprecated (2026-09-28 — superseded by stakeholder direction; see Addendum below)

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD §2 (Identity, Security & CI/CD): "GitLab CI/CD (`.gitlab-ci.yml`), Trivy (container vulnerability scanning)." PRD Epic 2 M4: "Trivy vulnerability scanning stage." PRD Epic 2 AC 3: "A merge to `main` triggers an automated, Trivy-scanned, zero-downtime rolling deployment via GitLab CI."

### Rationale
Directly satisfies Epic 2 AC 3's explicit "Trivy-scanned" requirement for Track A's pipeline. Positioned as a program-operational (Epic 4) ADR because it is the one CI/CD control named as a security gate for the whole delivery pipeline rather than a track-specific deployment mechanic, and it applies to the shared ECR images consumed by both tracks (Epic 3 M12 pulls the same images Track A's pipeline builds and scans).

### Consequences
**Positive**: Satisfies AC 3's exact requirement; catches known-vulnerable base images or dependencies before they reach ECR and, by extension, before Track B's ArgoCD/Helm path ever deploys them — a single scan gate protects both tracks' image supply chain.

**Negative**: A scan gate that fails the build on HIGH/CRITICAL findings can block a learner's pipeline on a transitive dependency vulnerability unrelated to their own code changes, which the guided-activity content needs to address explicitly (e.g., how to interpret and remediate a Trivy failure) so it does not become an unexplained blocker mid-module.

### Addendum (v6.0, 2026-09-28) — Deprecation

**Stakeholder direction**: "remove trivy scan as it is out of scope."

**This is a flagged conflict with the source PRD, not a PRD-driven change.** The PRD explicitly mandates Trivy in three places — §2 Tech Stack, Epic 2 M4 ("Trivy vulnerability scanning stage"), and Epic 2 AC 3 ("a Trivy-scanned... rolling deployment via GitLab CI") — none of which have been superseded by a PRD revision. Per this project's Grounding Rule 4 ("No BRD constraint may be violated without being explicitly flagged as a conflict"), this deviation is recorded here rather than silently applied.

**Effect on Track A pipeline** (see ADR-DCTA-AWS-E002-03): the GitLab CI pipeline for Track A drops the Trivy scan stage entirely. The pipeline becomes build → push to ECR → deploy (rolling, then Blue/Green in M6), with no image-vulnerability gate of any kind between build and deploy.

**Consequence of removal**: the image supply-chain protection described in this ADR's original "Positive" consequence above no longer exists for either track — Track A's ECR pushes and Track B's later consumption of the same images (Epic 3 M12) both go ungated. No compensating control has been requested or introduced. This should be re-raised with the PRD owner if the exclusion was not intended to also remove the AC 3 acceptance criterion itself, since AC 3 as written can no longer be satisfied by the Track A pipeline described in this document.

### Affected Epics
Primary: E004 — Program Operations, FinOps Guardrails & Capstone Defense
Also affects: E002 — Track A: AWS ECS & Fargate Path (the pipeline this gate ran in, see ADR-DCTA-AWS-E002-03)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — "CI/CD scanning — removed from scope"), §7.3 (CI/CD pipeline overview)

---

## ADR-DCTA-AWS-E004-03: Plain `terraform destroy` per stack, in dependency order, as the sole teardown mechanism

### Status
Accepted (revised 2026-09-28 — supersedes the v5.0 audit-script design; see Revision History below)

### Type
Decision — alternatives were evaluated

### Context
PRD Epic 4 AC 1 sets the absolute $5 ceiling. PRD Open Question 4 (resolved) confirms AWS Budgets alarms/SCPs enforcing this ceiling are assumed already configured on the pre-provisioned sandbox accounts, and that provisioning itself is out of this PRD's scope. PRD §6 Risks names the primary failure mode directly: "A learner forgets `terraform destroy`/`eksctl delete` and exceeds the $5 budget." Separately, Track B's teardown has a specific technical failure mode: deleting the EKS cluster stack before removing the ingress-nginx-provisioned Load Balancer (ADR-DCTA-AWS-E003-06) can orphan that Load Balancer and its ENIs, which block subnet deletion and continue billing even after the learner believes the session is torn down. Stakeholder direction (2026-09-28) asked why a custom teardown script was needed at all, and directed that teardown be plain `terraform destroy` only.

### Options Considered

#### Option 1: Rely solely on the pre-provisioned AWS Budgets alarms (80% alert, hard stop) named in the PRD's resolved Open Question 4, with no additional tooling or sequencing guidance
- **Pros**: Requires no additional tooling; matches the PRD's stated assumption that guardrails are already handled at the account level.
- **Cons**: Budgets alarms are a *detective* control — they fire after spend has already happened; on their own they say nothing about *how* a learner should run `terraform destroy` across four independent stacks (ADR-DCTA-AWS-E004-01), so a learner destroying stacks out of order can still hit the Track B orphaned-Load-Balancer failure mode with no guidance to prevent it.

#### Option 2: Plain `terraform destroy` run per Terraform stack, in a fixed dependency order, with the ingress-nginx `helm_release` resource configured with `wait = true` (and an adequate `timeout`) so its own destroy blocks until AWS has actually deprovisioned the Load Balancer it created — no separate script
- **Pros**: Uses only the tool already being taught (Terraform) and only the command the PRD's own SOP names (`terraform destroy`) — no new tool for learners to learn or for this design to maintain; the orphaned-Load-Balancer failure mode is closed *inside* Terraform itself rather than by an external script checking for it afterward, because `helm_release`'s `wait`/`timeout` arguments make the Kubernetes `Service` (and its AWS-provisioned Load Balancer) finish deleting before Terraform reports the `helm_release` resource as destroyed, which in turn is what unblocks destroying the stack that depends on it; teardown order follows directly from the stack dependency graph already established in ADR-DCTA-AWS-E004-01 (Track B: `41-track-b-platform` → `40-track-b-cluster` → `20-data`; Track A: `30-track-a` → `20-data`), so there is nothing extra to teach beyond "destroy stacks in reverse of the order you applied them."
- **Cons**: Depends on `wait = true` being correctly set (and a long-enough `timeout`) on every Terraform-managed resource that provisions an out-of-cluster AWS side effect — a future addition to the Track B platform stack that creates another cloud-managed resource (e.g., a second Load Balancer) would need the same treatment, and nothing in Terraform enforces that pattern automatically; gives a learner no positive, human-readable confirmation ("0 stray resources found") beyond Terraform's own destroy output and the AWS Budgets alarm — if a destroy silently fails partway through, the learner's first signal is the alarm, not an immediate local check.

### Decision
Option 2: plain `terraform destroy` on each of the four ephemeral stacks, run in dependency order, with `wait = true` set on the `ingress-nginx` `helm_release` resource (ADR-DCTA-AWS-E003-06) as the mechanism that makes the Load Balancer teardown ordering safe without any custom script.

### Rationale
Stakeholder direction explicitly asked why a bespoke teardown script was needed and directed that teardown be `terraform destroy` only — this overrides the v5.0 audit-script design even though the PRD itself does not prohibit additional tooling. Re-examining the orphaned-Load-Balancer risk that originally motivated the script (ADR-DCTA-AWS-E003-06 dependency) shows it is fully addressable inside Terraform: because the Load Balancer is a side effect of a Terraform-managed `helm_release` (not a resource created directly by kubectl or ArgoCD outside Terraform's view), Terraform can be made to wait for its actual deletion before proceeding, which removes the technical justification for an external verification script. Everything else ArgoCD manages lives inside the EKS cluster/node itself and is destroyed wholesale with the cluster — there is no additional out-of-cluster AWS resource in this architecture for a script to find that `terraform destroy` (with the `wait` fix) does not already handle.

### Consequences
**Positive**: No custom script to author, test, or keep in sync with the Terraform stacks as they evolve; teardown is exactly the command already named in the PRD's Daily FinOps SOP, run against a fixed, small, memorizable stack order; the Load-Balancer orphaning risk that motivated the original script is closed at its actual source (Terraform's own dependency tracking) rather than detected after the fact.

**Negative**: Loses the human-readable "0 stray resources found" confirmation the v5.0 `audit.sh` design gave learners and mentors at the end of a session — the only positive teardown signal now is a clean `terraform destroy` exit code per stack, backstopped solely by the AWS Budgets alarm if something is still missed; the safety of this approach rests entirely on `wait = true` + `timeout` being set correctly on every relevant `helm_release`, which must be verified whenever the Track B platform stack changes.

### Revision History
- **v5.0 (superseded)**: added a learner-run `audit.sh` script plus an explicit ordered teardown sequence (Argo `Application` cascade-delete → uninstall ingress-nginx → destroy platform stack → destroy cluster stack → destroy data stack), layered on top of the Budgets alarms.
- **v6.0 (current)**: removed `audit.sh` and the Argo cascade-delete step per stakeholder direction; teardown is plain `terraform destroy` per stack in dependency order, with the Load-Balancer-ordering risk closed via `helm_release wait = true` instead of a script.

### Affected Epics
Primary: E004 — Program Operations, FinOps Guardrails & Capstone Defense
Also affects: E003 — Track B: AWS EKS Path (the `wait = true` requirement applies to the ingress-nginx `helm_release` decided in ADR-DCTA-AWS-E003-06)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — "FinOps teardown mechanism"), §7.4 (Daily operating loop), §8 (NFR Fulfillment — Budget ceiling)

---

## ADR-DCTA-AWS-E004-04: Single-region, single-account operating model

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Technical Constraints: "Single region, single account model: no multi-region or multi-cloud deployments." PRD §5 System & Environment Requirements: "one isolated sandbox account (or equivalently isolated OU) per learner, region-locked to `ap-southeast-1`." PRD Epic 1 Out of Scope: "Multi-region or multi-cloud setup." PRD Epic 4 Out of Scope: "Sandbox account provisioning and issuance (accounts are provided to learners by the organization before the bootcamp begins)."

### Rationale
- Every Terraform stack in this design (ADR-DCTA-AWS-E004-01) targets exactly one AWS provider block, one region (`ap-southeast-1`), and one account — there is no cross-region replication, cross-account IAM trust, or multi-cloud provider configuration anywhere in the architecture, matching the PRD's explicit exclusion.
- This directly bounds the scope of the FinOps guardrails (ADR-DCTA-AWS-E004-03) and the Budgets alarms assumed in Open Question 4 to a single account per learner, which is what makes a simple dollar-ceiling alarm a sufficient detective control in the first place — a multi-account or multi-region design would need consolidated billing and cross-account cost aggregation this PRD does not call for.

### Consequences
**Positive**: Keeps every architectural decision in this document — network, compute, identity, delivery — scoped to one provider configuration, which is simpler to teach, simpler to tear down completely, and simpler to audit for stray billable resources (ADR-DCTA-AWS-E004-03) than a multi-account design would be.

**Negative**: Provides no isolation boundary beyond the account itself between a learner's own mistakes and the rest of their sandbox — there is no secondary account to fail over to or compare against; this is an accepted trade-off given that account provisioning and isolation are explicitly out of this PRD's scope and assumed handled by the organization beforehand.

### Affected Epics
Primary: E004 — Program Operations, FinOps Guardrails & Capstone Defense
Also affects: E001 — Shared Infrastructure Foundation; E002 — Track A: AWS ECS & Fargate Path; E003 — Track B: AWS EKS Path (all stacks operate within this single-region, single-account boundary)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §7.1 (Environments), §8 (NFR Fulfillment — Isolation / no cross-learner collision)

---

## ADR-DCTA-AWS-E004-05: GitLab hosting — gitlab.com (SaaS, shared runners)

### Status
Accepted

### Type
Decision — alternatives were evaluated

### Context
The PRD names GitLab CI/CD as the pipeline tool (§2 Tech Stack; Epic 2 M4; Epic 2 AC 3) but does not state where that GitLab instance is hosted. This was carried in v5.0 as Open Question 4: whether the bootcamp should target gitlab.com (SaaS) or a self-managed GitLab instance. Stakeholder direction (2026-09-28) resolved this directly: "gitlab is gitlab.com." The same direction also clarified that learners enter this bootcamp having already completed a prerequisite phase with the `devops-capstone-3tier-app` reference application, in which they already set the app up in Minikube and already have a working GitLab CI pipeline deploying to their local environment — meaning a GitLab project and pipeline already exist for these learners before this bootcamp's modules begin.

### Options Considered

#### Option 1: gitlab.com (SaaS, shared runners)
- **Pros**: No runner infrastructure to provision, patch, or tear down — directly serves the $5 budget ceiling (Epic 4 AC 1) by keeping CI/CD entirely off the metered AWS sandbox account; learners already have a gitlab.com account and project from the prerequisite `devops-capstone-3tier-app` phase (per stakeholder direction), so this bootcamp's CI/CD modules (M4, M14) extend an existing pipeline rather than standing up new hosting; shared runners are free at GitLab's standard SaaS tier for the pipeline complexity this PRD describes (build, push to ECR, trigger a deployment).
- **Cons**: Dependent on gitlab.com's shared-runner availability and any per-account CI/CD minute quotas, which this design does not control; less configuration control than a self-managed instance (e.g., custom runner tags, network-level access to the AWS sandbox VPC must go through public endpoints rather than a private runner).

#### Option 2: Self-managed GitLab (e.g., a GitLab instance run on an EC2 host or as a separate service)
- **Pros**: Full control over runner configuration, network placement (a runner could sit inside the learner's VPC), and CI/CD minute limits.
- **Cons**: A self-managed instance is itself a billable, always-on resource, which directly works against Epic 4 AC 1's $5 ceiling — there is no idle-cost-free way to keep a GitLab server available across a 2-week bootcamp within that budget; requires provisioning and maintaining GitLab itself, which is out of scope for a bootcamp about AWS containers, not GitLab administration; breaks the continuity with the prerequisite phase, where learners already have a gitlab.com project set up.

### Decision
Option 1: gitlab.com, using GitLab's shared SaaS runners. Both tracks' pipelines (Track A's build/push/deploy pipeline, Track B's GitOps-triggering pipeline) run here, extending the same gitlab.com project and pipeline the learner already has from the prerequisite `devops-capstone-3tier-app` phase rather than creating a new one.

### Rationale
Confirmed directly by stakeholder direction on 2026-09-28. The choice is also the only one of the two that keeps CI/CD hosting cost outside the $5 AWS sandbox budget entirely (Epic 4 AC 1), since gitlab.com's shared runners are not an AWS resource and do not appear in the learner's Cost Explorer at all — a self-managed instance would have to be either a separate, unbudgeted cost or an addition to the same $5 ceiling this design is built to protect. It is also the continuity-preserving choice: the stakeholder-confirmed assumption that learners already have a working local Minikube deployment and a working gitlab.com CI pipeline for the reference app means this bootcamp's CI/CD modules are additive to existing hosting, not a fresh setup.

### Consequences
**Positive**: Zero runner infrastructure for this design to provision or maintain; zero additional AWS spend from CI/CD hosting, preserving the full $5 ceiling for Epic 1–3 infrastructure; learners reuse an account and project they already have, reducing Module 4/14 setup time to pipeline-file changes rather than platform setup.

**Negative**: The design has no control over gitlab.com's shared-runner queue times or per-account CI/CD minute allowances, which could introduce variable pipeline latency outside this architecture's control; the AWS-side security-group/CIDR scoping decisions (ADR-DCTA-AWS-E001-05) must accommodate GitLab's public shared-runner IP ranges rather than a fixed, private runner address, since gitlab.com runners reach the AWS sandbox over the public internet.

### Affected Epics
Primary: E004 — Program Operations, FinOps Guardrails & Capstone Defense
Also affects: E002 — Track A: AWS ECS & Fargate Path (Track A's GitLab CI pipeline, ADR-DCTA-AWS-E002-03, runs on gitlab.com); E003 — Track B: AWS EKS Path (Track B's GitOps-triggering pipeline also runs on gitlab.com)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — "GitLab hosting"), §7.3 (CI/CD pipeline overview)
