# Retro / Backlog — VALIDATE 2026-09-28

Deferred items from the `/validate` pass on the DCTA-AWS curriculum
(modules/, capstone/, knowledge/, README). None block delivery.

## Backlog

1. **Live run not performed.** All five stacks pass `terraform validate`
   (providers aws 6.66.0, helm 3.3.0, random 3.9.1; EKS module 21.26.0), and
   the Helm solution templates pass `helm lint`/`helm template`, but nothing
   was `plan`ned or `apply`d against a real sandbox account. Instructor
   "expected output" tables are illustrative — replace them with captured
   output from one reference run before the first cohort.
2. **EKS KMS key per daily cluster.** `terraform-aws-modules/eks/aws` creates
   a KMS key by default (`create_kms_key = true`). It is tracked and
   scheduled for deletion on destroy, but a daily rebuild accumulates keys
   in "pending deletion". Consider `create_kms_key = false` + no
   `encryption_config` after testing that combination on module 21.26.0.
3. **AWS service-linked roles.** ECS, ELB, EKS, EKS node groups, Spot, RDS
   and Application Auto Scaling create service-linked roles on first use.
   They are account-level, free, and created by AWS (not the learner) — out
   of Terraform by design. Mention in instructor notes if learners ask.
4. **VPC CNI ENIs after an abrupt node loss.** If a Spot node is reclaimed
   mid-teardown, `aws-K8S-*` ENIs can linger and block subnet deletion on the
   last-day `10-foundation` destroy. Add a troubleshooting row if seen in a
   pilot.
5. **Two indented HCL fragments** (`lifecycle { … }` in the M4 and M6
   concept sections) intentionally fail stand-alone `terraform fmt -check`
   because they are shown at resource-body indentation. Gate 1 should
   exclude fragments that start with whitespace.
6. **Graphify graph is stale** (built before the module renames and
   regeneration). Run `graphify update .` so blast-radius queries see the
   new files.
7. **Track B node size** `t3.medium` is still pending sponsor approval
   (ADR-DCTA-AWS-E003-02).
8. **ARCH/PRD amendments to raise with the PRD owner:** ECS-native
   blue/green instead of CodeDeploy (E002-03, Epic 2 AC 5); F5 NGINX Ingress
   instead of ingress-nginx (E003-06); GitLab OIDC CI role instead of an IAM
   CI user (ARCH §6); Grafana host rule instead of `/grafana` path.

## Pattern contributed

- `knowledge/rules/coding-standards.md` → CI section: **Every AWS resource is
  created by Terraform**; CI uses GitLab OIDC, never access keys.

## /dev-tasks-planner on a curriculum repo (2026-09-28)

- The command's "scaffolded code must exist" prerequisite can't be met here:
  learners build in their own gitlab.com project, so there's no code to
  reconcile STATUS against. With the user's approval it ran as a **learner
  backlog**: `docs/dev-tasks/dcta-aws/track-{a,b}-tasks.csv`, every row
  `To Do`, ROLE `DevOps`, estimates taken from each track's Two-Week Timeline
  (10 days per track). Learners update STATUS in their own copy.
- The defense is **30 minutes** (rubric agendas rescaled, Q&A slot added);
  Track B keeps its Day 4 "rebuild from scratch" task.
