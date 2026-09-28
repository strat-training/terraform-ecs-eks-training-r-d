```
Project:      AWS Containers Bootcamp (Self-Paced, Hands-On)
Project Code: DCTA-AWS
Epic:         E001 — Shared Infrastructure Foundation
Version:      v6.0
Date:         2026-09-28
Source PRD:   PRD: AWS Containers Bootcamp (Self-Paced, Hands-On), v1.0, 2026-09-28
Source Arch:  ARCH-DCTA-AWS-v6.0
```

## ADR-DCTA-AWS-E001-01: Use Terraform as the Infrastructure-as-Code tool for the whole bootcamp

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD §2 Technical Stack: "Terraform, including the `terraform-aws-modules/eks/aws` community module (Track B) and the Terraform Helm Provider (`hashicorp/helm`)"; PRD §4 lists Terraform as required learner tooling from Day 1.

### Rationale
- Epic 1 AC 1–4: the VPC, RDS, Secrets Manager secret, and ECR repositories are all built and torn down through Terraform, so learners have one consistent tool for every module's infrastructure.
- Epic 4 Daily FinOps SOP: `terraform apply`/`terraform destroy` is the explicit mechanism the PRD names for the twice-daily cost-control loop; no other IaC tool is named anywhere in the syllabus.
- Epic 3 Tech Stack additionally names the `terraform-aws-modules/eks/aws` module and the Terraform Helm provider, so Terraform's role extends into Kubernetes-object management on Track B, not just AWS resources.

No alternative IaC tool (CloudFormation, CDK, Pulumi) is evaluated here because the PRD does not leave this open — it is a curriculum decision to teach Terraform specifically.

### Consequences
**Positive**: A single tool and mental model across both tracks and all 16 modules; the daily apply/destroy loop maps directly onto `terraform apply`/`terraform destroy` with no translation layer; the same skill (HCL, state, modules) compounds across the whole bootcamp.

**Negative**: Terraform state management (locking, drift, `terraform_remote_state` contracts between the split stacks in ADR-DCTA-AWS-E004-01) becomes an operational responsibility the curriculum must teach explicitly, since a corrupted or drifted state file with no environment ops team backing it up can strand a learner mid-session.

### Affected Epics
Primary: E001 — Shared Infrastructure Foundation
Also affects: E002 — Track A: AWS ECS & Fargate Path; E003 — Track B: AWS EKS Path; E004 — Program Operations, FinOps Guardrails & Capstone Defense (see ADR-DCTA-AWS-E004-01 for the stack-topology decision built on top of this choice)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Infrastructure as Code), §7.1 (Environments — Terraform stack table)

---

## ADR-DCTA-AWS-E001-02: Zero-NAT, 2-AZ VPC with public compute subnets and private data subnets

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 1 AC 1: "Custom VPC provisioned across 2 Availability Zones in `ap-southeast-1` with 2 public and 2 private subnets." PRD Technical Constraints: "No NAT Gateways: all ECS tasks and EKS worker nodes run in public subnets with public IPs to reach ECR at $0.00 NAT cost." PRD Epic 1 Out of Scope explicitly excludes NAT Gateways and Route 53.

### Rationale
- Epic 1 AC 1 fixes the exact subnet topology (2 public + 2 private, 2 AZs).
- The "no NAT" constraint is explicitly cost-motivated in the PRD's own Technical Constraints section, and it is the single largest recurring network cost this design avoids: a NAT Gateway is billed hourly per AZ plus per-GB data processing, and would consume a meaningful share of the $5 ceiling on its own if run across both AZs for the bootcamp's active hours.
- Compute (ECS tasks, EKS nodes) is placed in the public subnets specifically so it can reach ECR without a NAT path, satisfying the "$0.00 NAT cost" requirement directly.

### Consequences
**Positive**: Removes NAT Gateway cost entirely, which is one of the largest per-hour AWS network charges this workload could otherwise incur; keeps the design simple enough for a beginner-to-intermediate learner to reason about in one module (M1).

**Negative**: Every ECS task and EKS node carries a public IPv4 address, each of which is itself billed hourly (a smaller but nonzero cost that stacks with node/task count — see ADR-DCTA-AWS-E003-02 and the cost model in ARCH-DCTA-AWS-v6.0 §8); it also increases the network's exposed surface compared to a private-subnet-plus-NAT design, which is called out explicitly as a deliberate cost/pedagogy trade-off (not a production pattern) in Epic 1 and Epic 2/3's IAM modules.

### Affected Epics
Primary: E001 — Shared Infrastructure Foundation
Also affects: E002 — Track A: AWS ECS & Fargate Path (ECS tasks placed per this design); E003 — Track B: AWS EKS Path (EKS node placed per this design, see ADR-DCTA-AWS-E003-01)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Network design), §3.2 (Component table — VPC & subnets), §8 (NFR Fulfillment — Budget ceiling, Network exposure)

---

## ADR-DCTA-AWS-E001-03: Security-group chaining for the RDS access path

### Status
Accepted

### Type
Decision — alternatives were evaluated

### Context
PRD Epic 1 AC 2 requires RDS be "reachable only from the application tier on port 5432." Both the source (ALB/ECS tasks in Track A, EKS nodes in Track B) and the destination (RDS) are re-created daily under the Epic 4 FinOps SOP, so whatever mechanism enforces this restriction must survive that daily recreation without manual re-authoring.

### Options Considered

#### Option 1: CIDR-based security-group rules
- **Pros**: Simple to read for a beginner learner; no dependency ordering between security groups.
- **Cons**: The VPC's subnet CIDRs are stable, but a CIDR rule scoped to "the app-tier subnet" is looser than "the app-tier resources" — anything else later placed in that subnet would also gain DB access; does not teach the SG-reference pattern that is standard practice on AWS.

#### Option 2: Security-group-to-security-group references (SG chaining)
- **Pros**: The RDS SG's ingress rule references the app-tier SG by ID, not by CIDR, so it stays correct automatically no matter how many times ECS tasks or EKS nodes are destroyed and recreated with new IPs; scopes access to exactly the resources tagged with the app-tier SG, not merely "anything in this subnet"; is the AWS-recommended pattern and worth teaching as such.
- **Cons**: Requires the app-tier SG to exist before the RDS SG rule can reference it, adding a small amount of stack-ordering complexity across the `10-foundation`/`20-data`/`30-track-a`/`40-track-b-cluster` split (ADR-DCTA-AWS-E004-01).

### Decision
Use SG-to-SG references: an `alb-sg` (Track A) or `node-sg` (Track B) is allowed to reach an `app-sg`, which is in turn the only principal allowed to reach `rds-sg` on port 5432.

### Rationale
PRD Epic 1 AC 2's "reachable only from the application tier" is best satisfied by a rule that stays true regardless of the daily teardown/recreate cycle mandated by Epic 4 — a CIDR rule would need to be re-verified every session as ENIs and IPs change, while an SG reference does not. This also directly produces the acceptance-criteria evidence the PRD's own defense format calls for: a security-group rule visibly referencing another security group, not an IP range.

### Consequences
**Positive**: Correctness of the access boundary survives every daily teardown/recreate cycle without re-authoring; the pattern generalizes cleanly from Track A (ECS tasks in the app SG) to Track B (EKS nodes in the app SG), so the same RDS SG rule serves both tracks unmodified.

**Negative**: Introduces an explicit Terraform dependency edge (app-tier SG must exist before the RDS SG rule references it) that the split-stack topology (ADR-DCTA-AWS-E004-01) must expose as an output/input contract; a learner who deletes the app-tier SG without updating the RDS SG rule will get a less obvious Terraform error than with a plain CIDR rule.

### Affected Epics
Primary: E001 — Shared Infrastructure Foundation
Also affects: E002 — Track A: AWS ECS & Fargate Path; E003 — Track B: AWS EKS Path

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Security-group chaining), §6 (Integration and API Design — Backend → RDS row)

---

## ADR-DCTA-AWS-E001-04: RDS lifecycle — ephemeral by default, not always-on

### Status
Accepted

### Type
Decision — alternatives were evaluated

### Context
PRD Epic 1 Tech Stack names "Amazon RDS PostgreSQL (`db.t4g.micro`, Free Tier)." The PRD assumes Free Tier coverage but does not specify which Free Tier model the issued sandbox accounts fall under. AWS's Free Tier changed materially for accounts created on or after 2025-07-15: those accounts receive a shrinking credit balance ($100 + up to $100 more), not the legacy 750-free-hours-per-month RDS allowance. Sandbox accounts under an AWS Organization may also share allowances across learners. Left running continuously, a `db.t4g.micro` single-AZ PostgreSQL instance costs roughly $0.024/hour all-in (instance + 20 GB storage + 20 GB backup, `ap-southeast-1` planning rate) — around $8 over a 14-day bootcamp, which alone exceeds the PRD's entire $5.00 ceiling (§1 Goal 2) if the account is not Free-Tier-eligible.

### Options Considered

#### Option 1: Always-on RDS for the full 2 weeks
- **Pros**: Matches the mental model of "the database is always there," simplest to teach; no risk of RDS-endpoint-hostname churn between sessions; database seed data persists automatically.
- **Cons**: If the sandbox account is not on the legacy 750-hour model, this alone consumes more than the entire $5 budget before any compute is even created — a direct violation of Epic 4 AC 1 ("no learner's cumulative AWS spend... exceeds $5.00 USD at any point").

#### Option 2: Ephemeral RDS, created and destroyed with the daily session (default), with a documented persistent-mode override once Free Tier eligibility is confirmed
- **Pros**: Bounds RDS cost to actual lab hours regardless of the account's Free Tier model, which is the only choice that is safe under every eligibility scenario; consistent with the Epic 4 Daily FinOps SOP applied uniformly across every billable resource, not just compute.
- **Cons**: Adds ~5–10 minutes of daily session-start time for RDS to become available; requires the reference application to tolerate a fresh database on every session (idempotent migrations/seed — flagged as Open Question 5 in ARCH-DCTA-AWS-v6.0 §9), and requires confirming the RDS endpoint hostname stays predictable across recreates if a fixed `identifier` is used.

### Decision
Default to Option 2 (ephemeral RDS, created/destroyed with the daily `up.sh`/`down.sh` loop). Switch a learner's cohort to Option 1 only after the sandbox accounts' Free Tier model is confirmed by the program sponsor as the legacy 750-hour allowance (Open Question 1).

### Rationale
Epic 4 AC 1 is an absolute ceiling ("100% of learners"), not a target, so the default must be safe under the worst-case Free Tier scenario rather than the best case. The PRD's own resolved Open Question 4 confirms sandbox-account provisioning (and therefore its Free Tier model) is out of this PRD's scope and was set up by the organization beforehand — meaning this design cannot assume which model applies and must default to the option that is safe either way.

### Consequences
**Positive**: The $5 ceiling holds regardless of which Free Tier model the issued accounts turn out to have, closing the single largest identified budget risk in this design (see ARCH-DCTA-AWS-v6.0 §8 cost model); reinforces the pedagogical point (already present in the PRD's stateless/stateful decoupling framing) that state is a deliberate, managed choice, not an assumption.

**Negative**: Daily RDS creation adds session-start latency every single day across all 16 modules, not just once; if the reference application does not run idempotent migrations on start, an explicit seed/migration step must be added to the daily loop (Open Question 5), which is undecided as of this ADR.

### Affected Epics
Primary: E001 — Shared Infrastructure Foundation
Also affects: E002 — Track A: AWS ECS & Fargate Path; E003 — Track B: AWS EKS Path (both consume the same RDS instance); E004 — Program Operations, FinOps Guardrails & Capstone Defense (this decision is the largest single lever on the Epic 4 AC 1 budget ceiling)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Relational data store and lifecycle), §8 (NFR Fulfillment — Budget ceiling), §9 Open Question 1

---

## ADR-DCTA-AWS-E001-05: Split Secrets Manager (credentials only) from SSM Parameter Store (everything else)

### Status
Accepted

### Type
Decision — alternatives were evaluated

### Context
PRD Epic 1 AC 3 requires "Database credentials stored in AWS Secrets Manager, not hardcoded." The PRD names both Secrets Manager and SSM Parameter Store in §2 Technical Stack without specifying how configuration should be divided between them.

### Options Considered

#### Option 1: Put all configuration (secrets and non-secret values alike) into Secrets Manager
- **Pros**: One service to learn and reference; slightly simpler task-definition/ExternalSecret wiring since everything resolves the same way.
- **Cons**: Secrets Manager is billed per secret per month; every additional non-secret config value (hostnames, feature flags, region names) stored this way multiplies that per-secret charge against a $5 ceiling for no security benefit, since those values were never sensitive.

#### Option 2: One Secrets Manager secret for DB credentials; everything else in SSM Parameter Store (standard tier, free)
- **Pros**: Keeps exactly one billed secret in the whole system, satisfying Epic 1 AC 3 at minimum cost; SSM Parameter Store's standard tier carries no charge at this program's scale; teaches the correct distinction between "secret" and "configuration," which is itself a security-architecture lesson worth making explicit.
- **Cons**: Learners must learn two services instead of one, and two different injection mechanisms (task-def `secrets` vs. `environment`/SSM lookup in Track A; ExternalSecret vs. ConfigMap in Track B).

### Decision
Option 2: one Secrets Manager secret for DB credentials, SSM Parameter Store for all other configuration.

### Rationale
Epic 1 AC 3 only requires credentials to be in Secrets Manager — it does not require all configuration to live there. Given the $5.00 ceiling (§1 Goal 2), multiplying billed secrets for values that were never sensitive is a direct, avoidable cost with no corresponding requirement driving it.

### Consequences
**Positive**: Bounds Secrets Manager cost to one secret for the entire bootcamp; reinforces a security-architecture distinction (secret vs. configuration) that is good practice beyond this program.

**Negative**: Two services and two injection mechanisms to teach instead of one, adding a small amount of curriculum surface area in Module 1 and again when each track wires secrets/config into its compute layer (Track A task definitions in M2; Track B ExternalSecret/ConfigMap in M11–M12).

### Affected Epics
Primary: E001 — Shared Infrastructure Foundation
Also affects: E002 — Track A: AWS ECS & Fargate Path (task-def `secrets` wiring, M2); E003 — Track B: AWS EKS Path (ESO/`ExternalSecret` wiring, M11)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Secrets and configuration), §5 (Data Model), §6 (Integration and API Design — ESO → Secrets Manager row)

---

## ADR-DCTA-AWS-E001-06: Amazon ECR as the container registry, with a 2-image retention lifecycle policy

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 1 AC 4: "ECR repository holds pushed React and Node.js images." PRD §2 Technical Stack lists "Amazon ECR" under Cloud Platform & Infrastructure.

### Rationale
- Epic 1 AC 4 names ECR directly as the place both application images are pushed.
- Epic 1 is the persistent-foundation epic (ARCH-DCTA-AWS-v6.0 §3.1), and ECR repositories are created once in M1 and never torn down by the daily FinOps SOP, since ECR storage cost at this program's image count and size is a rounding error against the $5 ceiling — provided image history does not grow unbounded, which the lifecycle policy (keep last 2 tagged images per repository) prevents.

### Consequences
**Positive**: One registry, created once, reused by every module in both tracks and every learner session without recreation; the lifecycle policy keeps storage cost negligible for the life of the bootcamp without any learner action.

**Negative**: A learner who needs to roll back further than 2 image generations (e.g., to debug a regression introduced two deploys ago) cannot do so from ECR alone; this is an accepted trade-off given the program's 2-week scope and the negligible cost benefit of retaining more.

### Affected Epics
Primary: E001 — Shared Infrastructure Foundation
Also affects: E002 — Track A: AWS ECS & Fargate Path (GitLab CI pushes here, M4); E003 — Track B: AWS EKS Path (Helm/ArgoCD pull images from here, M12)

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Container registry), §3.2 (Component table — Amazon ECR), §7.4 (cost-avoidance pattern: "ECR lifecycle policy")
