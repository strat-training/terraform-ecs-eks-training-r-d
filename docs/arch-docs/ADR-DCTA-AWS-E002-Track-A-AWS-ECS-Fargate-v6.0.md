```
Project:      AWS Containers Bootcamp (Self-Paced, Hands-On)
Project Code: DCTA-AWS
Epic:         E002 — Track A: AWS ECS & Fargate Path
Version:      v6.0
Date:         2026-09-28
Source PRD:   PRD: AWS Containers Bootcamp (Self-Paced, Hands-On), v1.0, 2026-09-28
Source Arch:  ARCH-DCTA-AWS-v6.0
```

## ADR-DCTA-AWS-E002-01: ECS on Fargate/Fargate Spot with a 70/30 capacity split, at the smallest supported task size

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 2 Tech Stack: "Amazon ECS, AWS Fargate & Fargate Spot capacity providers." PRD Epic 2 M2: "70/30 Fargate Spot / On-Demand capacity provider split." PRD Epic 2 AC 1: "React and Node.js ECS services run on Fargate with a 70% Spot / 30% On-Demand capacity split."

### Rationale
- Satisfies Epic 2 AC 1 exactly as written.
- Fargate Spot offers up to a 70% discount off standard Fargate pricing; weighting 70% of desired capacity toward Spot materially reduces the per-live-hour compute cost that Epic 4 AC 1's $5 ceiling must absorb across two tasks (frontend, backend) running for the bulk of each Track A lab session.
- The smallest supported Fargate task size (0.25 vCPU / 0.5 GB) is chosen because the reference application (a task-manager CRUD app) does not need more, and Fargate is billed per vCPU-second and GB-second — the task size floor directly sets the compute cost floor.

### Consequences
**Positive**: Meets AC 1 exactly; keeps steady-state Track A compute cost to roughly the smallest achievable figure for a 2-task ECS deployment under Fargate pricing.

**Negative**: At 1 task (the steady state before Module 5's autoscaling), the 70/30 split is not observable — a single task is either Spot or On-Demand, not a blend — so the ratio can only be verified via the capacity-provider-strategy configuration until Module 5 scales to 3 tasks and a mixed capacity-type list becomes visible via `describe-tasks`. This is a known curriculum sequencing quirk, not a defect in the design.

### Affected Epics
Primary: E002 — Track A: AWS ECS & Fargate Path
Also affects: None

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track A compute), §3.2 (Component table — ECS cluster + Fargate/Fargate Spot)

---

## ADR-DCTA-AWS-E002-02: Single ALB with path-based routing, no Route 53

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 2 M3: "`/api/*` routed to backend target group, `/*` to frontend, single raw ALB DNS URL." PRD Epic 2 AC 2: "Single ALB path-routes `/api/*` to the backend and `/*` to the frontend with no Route 53 involved." PRD Epic 2 Out of Scope: "AWS Cloud Map / service discovery (ALB listener rules cover routing)." PRD Technical Constraints: "No Route 53 / custom domains: all testing uses raw ALB or Ingress/Load Balancer DNS URLs."

### Rationale
- Directly satisfies Epic 2 AC 2.
- One ALB serving both listener rules (instead of two load balancers, one per service) avoids a second ALB's hourly base charge and LCU cost against the $5 ceiling.
- Excluding Route 53 removes both its per-hosted-zone monthly charge and the need to teach DNS delegation inside a 2-week program whose defense criteria only require a reachable raw DNS name.

### Consequences
**Positive**: One load balancer serves the whole Track A capstone at minimum incremental cost; satisfies AC 2's routing requirement with a pattern (path-based rules on one ALB) that generalizes directly to Module 6's dual-target-group Blue/Green setup, since the ALB and its `/api/*` rule are reused, not replaced.

**Negative**: Evaluators and learners test against a raw, unmemorable ALB DNS name rather than a custom domain, which is an accepted pedagogy trade-off named explicitly in the PRD rather than a limitation introduced by this design.

### Affected Epics
Primary: E002 — Track A: AWS ECS & Fargate Path
Also affects: None

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track A traffic routing), §7.2 (Track A capstone deployment diagram)

---

## ADR-DCTA-AWS-E002-03: CI/CD deployment-mechanism transition — rolling update (M4) to CodeDeploy Blue/Green (M6)

### Status
Accepted

### Type
Decision — alternatives were evaluated

### Context
PRD Epic 2 M4 mandates "`aws ecs update-service` automated zero-downtime rolling deploy on merge to `main`." PRD Epic 2 M6 mandates "`CODE_DEPLOY` deployment controller, dual target groups, scripted Blue/Green cutover." Both mechanisms are explicitly required by the PRD, in that sequence, but they are not compatible on the same ECS service definition: once a service's deployment controller is set to `CODE_DEPLOY`, `aws ecs update-service` no longer performs the rollout, and switching a service's controller after creation requires the service to be replaced rather than updated in place. Left unaddressed, a learner following M4's pipeline literally into M6 would hit an unexplained failure at the exact moment they are learning the more advanced technique.

### Options Considered

#### Option 1: Treat M6 as a configuration change to the existing M4 pipeline and service
- **Pros**: Feels like a natural, incremental extension to a learner; less new Terraform/CI content to author.
- **Cons**: Factually incorrect — `CODE_DEPLOY` controller adoption requires recreating the ECS service, not just changing a CI step; presenting it as an in-place change would produce a broken guided activity and an unexplained failure right before the capstone module.

#### Option 2: Treat M6 as an explicit refactor — the backend ECS service is deliberately recreated with the `CODE_DEPLOY` controller, and the CI pipeline's deploy step is swapped from `aws ecs update-service` to `aws deploy create-deployment`
- **Pros**: Matches the actual AWS mechanics, so the guided activity and unguided challenge are both technically accurate; gives learners an explicit lesson in *ownership hand-off* (which tool controls which field, and when that control changes) that generalizes to the Track B equivalent (Helm → ArgoCD in M14); the frontend service is deliberately left on ECS rolling deploys, so the contrast between the two deployment strategies is visible side by side in the same capstone.
- **Cons**: Requires explicit curriculum content calling out the service-replacement step and the CI pipeline branch/stage change; slightly larger scope for M6's guided activity than a pure "add Blue/Green" framing would suggest.

### Decision
Option 2. Module 6 is authored as an explicit refactor of the Module 2/Module 4 backend service and pipeline, not an incremental add-on, and only the backend service adopts `CODE_DEPLOY` — the frontend keeps its ECS rolling deployment.

### Rationale
PRD Epic 2 AC 5 requires "A live Blue/Green deployment completes successfully via CodeDeploy" as a defense-time acceptance criterion; a design that silently breaks at the M4→M6 boundary would put that at risk on the one day it matters most (the live defense, Epic 2 AC 7). Naming the ownership hand-off explicitly (documented in ARCH-DCTA-AWS-v6.0 §4.4-equivalent ownership table) turns a potential point of learner confusion into a deliberate teaching point about deployment-controller ownership, consistent with the PRD's own framing of the bootcamp as teaching production-realistic AWS patterns.

### Consequences
**Positive**: The M6 guided activity and unguided challenge are technically accurate, avoiding a failure mode that would otherwise surface for every learner and every cohort; produces a clean, demonstrable contrast (rolling frontend vs. Blue/Green backend) for the live defense narration required by AC 7.

**Negative**: M6 carries more curriculum weight than a simple "enable Blue/Green" framing — it must explicitly teach service recreation and a CI pipeline branch change, adding authoring effort beyond what the PRD's one-line feature description implies.

### Affected Epics
Primary: E002 — Track A: AWS ECS & Fargate Path
Also affects: None

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track A CI/CD and deployment mechanism), §7.3 (CI/CD pipeline overview)

---

## ADR-DCTA-AWS-E002-04: CloudWatch logs, dashboard, and Application Auto Scaling target-tracking

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 2 M5: "`awslogs` log routing, CloudWatch dashboards, target-tracking auto-scaling (1–3 tasks at 70% CPU)." PRD Epic 2 AC 4: "Backend auto-scales 1→3 tasks on CPU > 70%, with logs visible in CloudWatch."

### Rationale
Directly satisfies Epic 2 AC 4. CloudWatch is the default logging/metrics destination for ECS (`awslogs` driver) and requires no additional service to stand up, keeping this module's incremental cost to log-storage and dashboard usage, both of which are sized to stay inside AWS's free allowances at this program's scale (short retention, no Container Insights, per the cost-avoidance patterns in ARCH-DCTA-AWS-v6.0 §7).

### Consequences
**Positive**: Satisfies AC 4 with no new AWS service beyond what ECS already integrates with by default; autoscaling logic (1→3 tasks, 70% CPU) is declared once in Terraform and requires no application-code changes.

**Negative**: Short log retention (kept low deliberately for cost) limits how far back a learner can debug a failed deployment from a prior session, since Track A's ephemeral compute means the log group itself may not have existed before the current session's `up.sh`.

### Affected Epics
Primary: E002 — Track A: AWS ECS & Fargate Path
Also affects: None

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track A observability and scaling), §8 (NFR Fulfillment — Scalability, Observability)

---

## ADR-DCTA-AWS-E002-05: Separate IAM Task Execution Role from App Task Role, scoped to one secret ARN

### Status
Accepted

### Type
Prescribed — mandated by PRD — not open for evaluation

### Source
PRD Epic 2 M7: "Separate Task Execution Role from App Task Role; backend scoped to its own Secrets Manager / Parameter Store ARN only." PRD Epic 2 AC 6: "Task Roles are scoped so the backend can read only its own secret, verified by a denied test call."

### Rationale
- Satisfies Epic 2 AC 6 directly, including its specific verification method (a denied test call against a second, out-of-scope secret ARN).
- The split itself (Execution Role vs. Task Role) is the AWS-standard least-privilege pattern for ECS: the Execution Role is what the ECS agent uses to pull the image, inject secrets, and write logs; the Task Role is what the running application code may call. Conflating the two would give application code broader AWS permissions than it needs — exactly what AC 6's test is designed to catch if done wrong.

### Consequences
**Positive**: Satisfies AC 6's exact acceptance test; teaches the least-privilege split that is standard ECS practice, reusable knowledge beyond this program; narrows blast radius if application code is ever compromised, since the Task Role cannot read secrets outside its own ARN.

**Negative**: Adds a second IAM role and a second policy document to author and review per service, beyond the single-role model a learner might reach for first; the M7 unguided challenge specifically requires learners to construct and pass a negative test (an intentionally denied call), which is more curriculum surface than a positive-only test would need.

### Affected Epics
Primary: E002 — Track A: AWS ECS & Fargate Path
Also affects: None

### Architecture Document Reference
ARCH-DCTA-AWS-v6.0 §4 (Technology Choices — Track A identity), §8 (NFR Fulfillment — Security, least privilege)
