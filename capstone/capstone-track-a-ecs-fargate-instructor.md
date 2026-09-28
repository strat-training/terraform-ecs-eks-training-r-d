# Capstone — Track A: Task Manager on ECS Fargate (Instructor Version)

In this capstone the cohort brings together M1–M7: the task-manager app on
**Amazon ECS with Fargate**, behind **one Application Load Balancer**,
deployed by **GitLab CI** with **blue/green** releases for the backend,
watched by **CloudWatch**, auto-scaled, and locked down with
**least-privilege IAM** — all in **Terraform**, torn down every session.

> **Disclaimer**
>
> The TaskFlow team is fictional. Built for the DevOps Bootcamp's
> *Containerization in AWS* phase (DCTA-AWS), using the reference app
> `devops-capstone-3tier-app`.

> **Instructor-only content.** This file contains the reference solution,
> expected outputs and checkpoint answers. Do not share it with learners —
> give them `capstone-track-a-ecs-fargate-cohort.md`.

## Getting started

### Prerequisites

Same as the cohort version: sandbox account in `ap-southeast-1`, the state
bucket name, the learner's gitlab.com project with the M1 app changes,
Terraform 1.11+, AWS CLI v2, Docker `buildx`, `git`, `curl`, `jq`, `ab`.

### Local setup

Use this to walk a reference build while validating a submission. All
commands run from the repo root.

```bash
# Settings and today's IP
source infra/env.sh
export TF_VAR_learner_cidr="$(curl -fsS https://checkip.amazonaws.com)/32"

# Newest images already in ECR
export TF_VAR_frontend_image_tag="$(aws ecr describe-images --repository-name dcta-frontend --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags[0]' --output text)"
export TF_VAR_backend_image_tag="$(aws ecr describe-images --repository-name dcta-backend --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags[0]' --output text)"

# Apply in dependency order
terraform -chdir=infra/10-foundation init -backend-config=../backend.hcl
terraform -chdir=infra/10-foundation apply
terraform -chdir=infra/20-data init -backend-config=../backend.hcl
terraform -chdir=infra/20-data apply
terraform -chdir=infra/30-track-a init -backend-config=../backend.hcl
terraform -chdir=infra/30-track-a apply

# Seed and check exit code
TASK_ARN="$(eval "$(terraform -chdir=infra/30-track-a output -raw seed_run_command)")"
aws ecs wait tasks-stopped --cluster dcta-ecs --tasks "$TASK_ARN"
aws ecs describe-tasks --cluster dcta-ecs --tasks "$TASK_ARN" --query 'tasks[0].containers[0].exitCode'
```

Verification — one happy path, one failure path:

```bash
ALB="$(terraform -chdir=infra/30-track-a output -raw alb_dns_name)"

# Happy path: 200 and a JSON list of 3 seed tasks
curl -s -o /dev/null -w '%{http_code}\n' "http://$ALB/"
curl -s "http://$ALB/api/tasks" | jq length

# Failure path: backend at 0 -> /api/tasks 503, / still 200
aws ecs update-service --cluster dcta-ecs --service dcta-backend --desired-count 0 >/dev/null
sleep 60
curl -s -o /dev/null -w '%{http_code}\n' "http://$ALB/api/tasks"
curl -s -o /dev/null -w '%{http_code}\n' "http://$ALB/"
aws ecs update-service --cluster dcta-ecs --service dcta-backend --desired-count 1 >/dev/null
```

Teardown:

```bash
terraform -chdir=infra/30-track-a destroy
terraform -chdir=infra/20-data destroy
aws elbv2 describe-load-balancers --query 'length(LoadBalancers)'
```

Last day only — remove the task definition revisions CI registered (not in
Terraform state), then the foundation:

```bash
for arn in $(aws ecs list-task-definitions --family-prefix dcta- --status ACTIVE --query 'taskDefinitionArns[]' --output text); do
  aws ecs deregister-task-definition --task-definition "$arn" > /dev/null
done
for arn in $(aws ecs list-task-definitions --family-prefix dcta- --status INACTIVE --query 'taskDefinitionArns[]' --output text); do
  aws ecs delete-task-definitions --task-definitions "$arn" > /dev/null
done
terraform -chdir=infra/10-foundation destroy
```

> Note: once auto scaling owns `desired_count` (M5), the manual scale-to-0
> test may be undone by the scaling policy's minimum of 1 within a minute or
> two. Run it quickly, or run it before M5 is applied.

### Data setup

No external data. The reference app's `app/database/init.sql` was rewritten
in M1 as the idempotent `app/database/seed.sql` (see **Data sources**).
The `dcta-db-seed` task loads it each session.

## Business problem

> Can a small team run its task-manager app on AWS so that releases are
> safe and automatic, the app handles busy moments by itself, and nothing
> can reach the database or its password except the app — while the whole
> environment can be switched off at the end of every day?

- Release by merging to `main`, test privately, switch back instantly.
- Scale the backend automatically during busy periods.
- Prove each identity can do only its job.
- Pay for infrastructure only while someone is working.

## Requirements / acceptance criteria

Each requirement is followed by **how the reference solution satisfies
it** (instructor-only). Solution snippets are in **Key engineering features**.

1. **VPC, 2 AZs, 2 public + 2 private subnets, IGW, no NAT (M1).**
   *Solution:* `aws_vpc` `10.20.0.0/16`; `aws_subnet.public[0..1]` via
   `cidrsubnet(…, 8, 0..1)` with `map_public_ip_on_launch = true`;
   `aws_subnet.private[0..1]` via `cidrsubnet(…, 8, 10..11)`; one public
   route table to the IGW; private subnets stay on the main (local-only)
   route table. No `aws_nat_gateway` resource exists.
2. **SG chain with `/32` ingress; RDS only from app SG (M1).**
   *Solution:* `aws_vpc_security_group_ingress_rule` resources: ALB `80`
   from `var.learner_cidr`; app `80`/`3000` from `referenced_security_group_id
   = alb`; app self-reference (Track B node ↔ pod); RDS `5432` from app.
   ALB egress only to app SG; app egress `0.0.0.0/0` with justification.
3. **Stacks, S3 state with locking, exact pins, remote state (M1).**
   *Solution:* `backend "s3" { key = … }` + `backend.hcl` with
   `use_lockfile = true`, `encrypt = true`; `version = "6.66.0"` /
   `"3.9.1"`; `data "terraform_remote_state"` in `20-data` and `30-track-a`.
4. **ECR immutable, keep last 2, `amd64`, SHA tags (M1).**
   *Solution:* given in M1 lab step 4; images pushed with
   `docker buildx --platform linux/amd64` and `$(git rev-parse --short HEAD)`.
5. **Credentials in Secrets Manager, none in Git/state (M1).**
   *Solution:* `ephemeral "random_password"` → `secret_string_wo`;
   `20-data` reads it with `ephemeral "aws_secretsmanager_secret_version"`
   → `password_wo`. Verify with `terraform state pull | grep -c password`
   → only the attribute names/versions, no value.
6. **RDS private, re-creatable; `nc` fails (M1).**
   *Solution:* DB subnet group on private subnets, `publicly_accessible =
   false`, `skip_final_snapshot = true`, `backup_retention_period = 0`,
   SSM `/dcta/db/host` from `aws_db_instance.postgres.address`.
7. **Task definitions (M2).** *Solution:* see snippet **A-2**.
8. **70/30 Spot; seed task re-runnable (M2).** *Solution:* two
   `capacity_provider_strategy` blocks (7/3); seed task given in M2 lab.
9. **`/api/*` rule; 503/200 failure proof (M3).** *Solution:* snippet **A-3**.
10. **CI build/push/deploy, scoped identity, waits for stable (M4).**
    *Solution:* GitLab OIDC provider + `dcta-gitlab-ci` role in the M4 lab (trust pinned to `project_path:<group>/<project>:ref_type:branch:ref:main`); deploy jobs in snippet **A-4**.
11. **Zero-downtime frontend; broken image rolls back (M4).**
    *Solution:* `deployment_minimum_healthy_percent = 100`,
    `deployment_maximum_percent = 200`, `deployment_circuit_breaker {
    enable = true, rollback = true }`.
12. **Backend blue/green, test route, 5-min bake; rollback (M6).**
    *Solution:* snippet **A-6**.
13. **Logs ≤ 7 days, dashboard (M5).** *Solution:* log groups with
    `retention_in_days = 3`; dashboard given in M5 lab.
14. **Scaling 1 → 3 at 70%; Spot mix visible (M5).** *Solution:* snippet **A-5**.
15. **Scoped task roles; probe denials (M7).** *Solution:* snippet **A-7**.
16. **30-minute defense, verified teardown (M8).** *Solution:* rubric
    agenda; destroy `30-track-a` → `20-data`; `describe-load-balancers` → `0`.

## Deliverable

A running system plus evidence (non-UI deliverable): the app at the ALB
address, the gitlab.com pipeline, CloudWatch log groups/dashboard/scaling
activity, and `docs/capstone/` in the learner's repo.

A correct session log looks like this (illustrative — capture a real run
when validating the reference build):

```text
Apply complete! Resources: 34 added, 0 changed, 0 destroyed.   # 30-track-a
"0"                                                             # seed exit code
200                                                             # GET /
3                                                               # jq length /api/tasks
```

## Data sources

| Source | Origin | Type |
|---|---|---|
| Task schema + 3 sample tasks | Reference app `app/database/init.sql` → `seed.sql` (M1) | Static, bounded |
| Demo tasks | Created live through the UI/API | Live, one session |
| Failure inputs | Backend at 0, broken image, forbidden secret | Deliberate |

## Project architecture

```text
Developer ──git push──► gitlab.com CI ──build/push──► ECR
                              └──register task def + update-service──► ECS
Browser (/32) ──► ALB :80 ─┬─ rule 90  /api/* + X-Deploy-Stage: test ─► backend alt TG (green)
                           ├─ rule 100 /api/* ─────────────────────────► backend TG (blue ⇄ green)
                           └─ default ─────────────────────────────────► frontend TG (rolling)
backend ──TLS :5432──► RDS (private)   backend ◄── secrets ── Secrets Manager / SSM
tasks ──awslogs──► CloudWatch ──► dashboard;  CPU ──► App Auto Scaling (1–3 @ 70%)
```

Orchestration: the learner (daily apply/destroy), GitLab CI (merges), ECS
+ Application Auto Scaling (continuous).

## Data model

`tasks` (UUID `id`, `title`, `description`, `status` in
`TODO|IN_PROGRESS|DONE`, `created_at`, `updated_at` via trigger).

## Technology stack

| Area | Technology | Module |
|---|---|---|
| IaC | Terraform 1.11+, `hashicorp/aws` 6.66.0, `hashicorp/random` 3.9.1, S3 state + native lock | M1 |
| Network | VPC, IGW, chained SGs | M1 |
| Registry | ECR | M1 |
| Database | RDS PostgreSQL 17 `db.t4g.micro` | M1 |
| Secrets | Secrets Manager, SSM Parameter Store | M1 |
| Compute | ECS Fargate + Fargate Spot | M2 |
| Traffic | ALB | M2–M3 |
| CI/CD | gitlab.com CI (`docker:29`, `docker:29-dind`) | M4 |
| Observability | CloudWatch Logs, dashboard | M5 |
| Scaling | Application Auto Scaling | M5 |
| Releases | ECS blue/green | M6 |
| IAM | Execution role, task roles, ECS infrastructure role | M2, M6, M7 |

## Key engineering features

Reference solution snippets. Stack skeletons (`versions.tf`,
`variables.tf`, `remote_state.tf`, `iam.tf` execution role, `alb.tf`,
`seed.tf`, `ci.tf`, `dashboard.tf`, `bluegreen.tf`, `iam_probe.tf`) are
exactly as given in the module labs.

**A-1 — `infra/10-foundation/vpc.tf` (M1 exercise)**

```hcl
locals {
  azs = ["ap-southeast-1a", "ap-southeast-1b"]
}

resource "aws_vpc" "main" {
  cidr_block           = "10.20.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags                 = { Name = "dcta-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "dcta-igw" }
}

resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true
  tags = {
    Name                     = "dcta-public-${local.azs[count.index]}"
    "kubernetes.io/role/elb" = "1"
  }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 10)
  availability_zone = local.azs[count.index]
  tags              = { Name = "dcta-private-${local.azs[count.index]}" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "dcta-public-rt" }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

**A-1 — `infra/10-foundation/security_groups.tf`**

```hcl
resource "aws_security_group" "alb" {
  name        = "dcta-alb-sg"
  description = "Track A ALB: HTTP from the learner only"
  vpc_id      = aws_vpc.main.id
}

resource "aws_security_group" "app" {
  name        = "dcta-app-sg"
  description = "App tier: ECS tasks or EKS node"
  vpc_id      = aws_vpc.main.id
}

resource "aws_security_group" "rds" {
  name        = "dcta-rds-sg"
  description = "RDS: 5432 from the app tier only"
  vpc_id      = aws_vpc.main.id
}

resource "aws_vpc_security_group_ingress_rule" "alb_http" {
  security_group_id = aws_security_group.alb.id
  cidr_ipv4         = var.learner_cidr
  from_port         = 80
  to_port           = 80
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "alb_to_app" {
  security_group_id            = aws_security_group.alb.id
  referenced_security_group_id = aws_security_group.app.id
  ip_protocol                  = "-1"
}

resource "aws_vpc_security_group_ingress_rule" "app_from_alb" {
  for_each                     = toset(["80", "3000"])
  security_group_id            = aws_security_group.app.id
  referenced_security_group_id = aws_security_group.alb.id
  from_port                    = tonumber(each.value)
  to_port                      = tonumber(each.value)
  ip_protocol                  = "tcp"
}

resource "aws_vpc_security_group_ingress_rule" "app_self" {
  security_group_id            = aws_security_group.app.id
  referenced_security_group_id = aws_security_group.app.id
  ip_protocol                  = "-1"
}

resource "aws_vpc_security_group_egress_rule" "app_all" {
  security_group_id = aws_security_group.app.id
  cidr_ipv4         = "0.0.0.0/0" # justification: no NAT; tasks/nodes reach ECR and AWS APIs directly
  ip_protocol       = "-1"
}

resource "aws_vpc_security_group_ingress_rule" "rds_from_app" {
  security_group_id            = aws_security_group.rds.id
  referenced_security_group_id = aws_security_group.app.id
  from_port                    = 5432
  to_port                      = 5432
  ip_protocol                  = "tcp"
}
```

**A-1 — `infra/10-foundation/outputs.tf`**

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "alb_sg_id" {
  value = aws_security_group.alb.id
}

output "app_sg_id" {
  value = aws_security_group.app.id
}

output "rds_sg_id" {
  value = aws_security_group.rds.id
}

output "ecr_repository_urls" {
  value = {
    frontend = aws_ecr_repository.app["dcta-frontend"].repository_url
    backend  = aws_ecr_repository.app["dcta-backend"].repository_url
  }
}

output "db_secret_arn" {
  value = aws_secretsmanager_secret.db.arn
}

output "db_name_parameter_arn" {
  value = aws_ssm_parameter.db_name.arn
}
```

**A-1 — `infra/20-data/main.tf`** (plus `versions.tf` with key
`dcta/20-data.tfstate`, variables, and a `terraform_remote_state` read of
`10-foundation` into `local.f`; the `aws_db_instance` is the M1 lab sample)

```hcl
resource "aws_db_subnet_group" "private" {
  name       = "dcta-db-private"
  subnet_ids = local.f.private_subnet_ids
}

resource "aws_ssm_parameter" "db_host" {
  name  = "/dcta/db/host"
  type  = "String"
  value = aws_db_instance.postgres.address
}

output "db_host_parameter_arn" {
  value = aws_ssm_parameter.db_host.arn
}
```

**A-2 — task definitions and services (`infra/30-track-a/services.tf`)**

```hcl
variable "frontend_image_tag" {
  type = string
}

variable "backend_image_tag" {
  type = string
}

resource "aws_cloudwatch_log_group" "app" {
  for_each          = toset(["frontend", "backend"])
  name              = "/dcta/ecs/${each.value}"
  retention_in_days = 3
}

resource "aws_ecs_task_definition" "frontend" {
  family                   = "dcta-frontend"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.execution.arn
  task_role_arn            = aws_iam_role.frontend_task.arn

  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "X86_64"
  }

  container_definitions = jsonencode([{
    name         = "frontend"
    image        = "${local.f.ecr_repository_urls.frontend}:${var.frontend_image_tag}"
    essential    = true
    portMappings = [{ containerPort = 80, protocol = "tcp" }]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = aws_cloudwatch_log_group.app["frontend"].name
        awslogs-region        = var.region
        awslogs-stream-prefix = "frontend"
      }
    }
  }])
}

resource "aws_ecs_task_definition" "backend" {
  family                   = "dcta-backend"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.execution.arn
  task_role_arn            = aws_iam_role.backend_task.arn

  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "X86_64"
  }

  container_definitions = jsonencode([{
    name         = "backend"
    image        = "${local.f.ecr_repository_urls.backend}:${var.backend_image_tag}"
    essential    = true
    portMappings = [{ containerPort = 3000, protocol = "tcp" }]
    environment = [
      { name = "PORT", value = "3000" },
      { name = "DB_PORT", value = "5432" },
      { name = "DB_SSL_CA_PATH", value = "/app/certs/rds-ca.pem" }
    ]
    secrets = [
      { name = "DB_USER", valueFrom = "${local.f.db_secret_arn}:username::" },
      { name = "DB_PASSWORD", valueFrom = "${local.f.db_secret_arn}:password::" },
      { name = "DB_HOST", valueFrom = local.d.db_host_parameter_arn },
      { name = "DB_NAME", valueFrom = local.f.db_name_parameter_arn }
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = aws_cloudwatch_log_group.app["backend"].name
        awslogs-region        = var.region
        awslogs-stream-prefix = "backend"
      }
    }
  }])
}

resource "aws_ecs_service" "frontend" {
  name                               = "dcta-frontend"
  cluster                            = aws_ecs_cluster.this.id
  task_definition                    = aws_ecs_task_definition.frontend.arn
  desired_count                      = 1
  health_check_grace_period_seconds  = 60
  deployment_minimum_healthy_percent = 100
  deployment_maximum_percent         = 200

  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 7
  }

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 3
  }

  network_configuration {
    subnets          = local.f.public_subnet_ids
    security_groups  = [local.f.app_sg_id]
    assign_public_ip = true
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.frontend.arn
    container_name   = "frontend"
    container_port   = 80
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  depends_on = [aws_lb_listener.http, aws_ecs_cluster_capacity_providers.this]

  lifecycle {
    ignore_changes = [task_definition, desired_count]
  }
}
```

**A-3 — listener rules (`infra/30-track-a/rules.tf`)** — the M3 rule, plus
the M6 test rule

```hcl
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.backend.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }

  lifecycle {
    ignore_changes = [action] # ECS blue/green owns the forward target
  }
}

resource "aws_lb_listener_rule" "api_test" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 90

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.backend_alt.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }

  condition {
    http_header {
      http_header_name = "X-Deploy-Stage"
      values           = ["test"]
    }
  }

  lifecycle {
    ignore_changes = [action]
  }
}
```

**A-4 — CI deploy jobs (`.gitlab-ci.yml`, added to the M4 build jobs)**

```yaml
.aws-deploy:
  stage: deploy
  image: docker:29
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: https://gitlab.com
  before_script:
    - apk add --no-cache aws-cli jq
    - echo "$GITLAB_OIDC_TOKEN" > /tmp/web-identity-token
    - export AWS_WEB_IDENTITY_TOKEN_FILE=/tmp/web-identity-token AWS_ROLE_SESSION_NAME="gitlab-${CI_PIPELINE_ID}"
    - export ECR_REGISTRY="$(aws sts get-caller-identity --query Account --output text).dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
  script:
    - |
      STATUS="$(aws ecs describe-services --cluster dcta-ecs --services "$SERVICE" --query 'services[0].status' --output text)"
      if [ "$STATUS" != "ACTIVE" ]; then echo "Service not running (stack destroyed); image pushed, skipping deploy"; exit 0; fi
    - IMAGE="$ECR_REGISTRY/$SERVICE:$IMAGE_TAG"
    - |
      aws ecs describe-task-definition --task-definition "$SERVICE" --query taskDefinition \
        | jq --arg IMG "$IMAGE" '.containerDefinitions[0].image = $IMG
            | del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)' \
        > taskdef.json
    - REV="$(aws ecs register-task-definition --cli-input-json file://taskdef.json --query taskDefinition.taskDefinitionArn --output text)"
    - aws ecs update-service --region ap-southeast-1 --cluster dcta-ecs --service "$SERVICE" --task-definition "$REV" > /dev/null
    - aws ecs wait services-stable --cluster dcta-ecs --services "$SERVICE"

deploy-backend-aws:
  extends: .aws-deploy
  needs: ["build-backend-aws"]
  variables:
    SERVICE: dcta-backend

deploy-frontend-aws:
  extends: .aws-deploy
  needs: ["build-frontend-aws"]
  variables:
    SERVICE: dcta-frontend
```

Notes: task-definition families and service names are both `dcta-<name>`,
so `$SERVICE` doubles as the family. `services-stable` polls for up to 10
minutes — enough for a 5-minute blue/green bake plus start-up; if learners
raise the bake time they must also split the wait.

**A-5 — auto scaling (`infra/30-track-a/autoscaling.tf`)**

```hcl
resource "aws_appautoscaling_target" "backend" {
  service_namespace  = "ecs"
  scalable_dimension = "ecs:service:DesiredCount"
  resource_id        = "service/${aws_ecs_cluster.this.name}/${aws_ecs_service.backend.name}"
  min_capacity       = 1
  max_capacity       = 3
}

resource "aws_appautoscaling_policy" "backend_cpu" {
  name               = "dcta-backend-cpu-70"
  policy_type        = "TargetTrackingScaling"
  service_namespace  = aws_appautoscaling_target.backend.service_namespace
  scalable_dimension = aws_appautoscaling_target.backend.scalable_dimension
  resource_id        = aws_appautoscaling_target.backend.resource_id

  target_tracking_scaling_policy_configuration {
    target_value       = 70
    scale_out_cooldown = 60
    scale_in_cooldown  = 180

    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
  }
}
```

**A-6 — backend service with blue/green (`infra/30-track-a/services.tf`)**

```hcl
resource "aws_ecs_service" "backend" {
  name                               = "dcta-backend"
  cluster                            = aws_ecs_cluster.this.id
  task_definition                    = aws_ecs_task_definition.backend.arn
  desired_count                      = 1
  health_check_grace_period_seconds  = 60
  deployment_minimum_healthy_percent = 100
  deployment_maximum_percent         = 200

  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 7
  }

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 3
  }

  network_configuration {
    subnets          = local.f.public_subnet_ids
    security_groups  = [local.f.app_sg_id]
    assign_public_ip = true
  }

  deployment_configuration {
    strategy             = "BLUE_GREEN"
    bake_time_in_minutes = 5
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.backend.arn
    container_name   = "backend"
    container_port   = 3000

    advanced_configuration {
      alternate_target_group_arn = aws_lb_target_group.backend_alt.arn
      production_listener_rule   = aws_lb_listener_rule.api.arn
      test_listener_rule         = aws_lb_listener_rule.api_test.arn
      role_arn                   = aws_iam_role.ecs_lb_infra.arn
    }
  }

  depends_on = [aws_ecs_cluster_capacity_providers.this, aws_iam_role_policy_attachment.ecs_lb_infra]

  lifecycle {
    ignore_changes = [task_definition, desired_count, load_balancer]
  }
}
```

**A-7 — task roles (`infra/30-track-a/task_roles.tf`)**

```hcl
resource "aws_iam_role" "backend_task" {
  name               = "dcta-backend-task"
  assume_role_policy = data.aws_iam_policy_document.ecs_tasks_assume.json
}

data "aws_iam_policy_document" "backend_task" {
  statement {
    actions   = ["secretsmanager:GetSecretValue"]
    resources = [local.f.db_secret_arn]
  }

  statement {
    actions   = ["ssm:GetParameter", "ssm:GetParameters"]
    resources = [local.f.db_name_parameter_arn, local.d.db_host_parameter_arn]
  }
}

resource "aws_iam_role_policy" "backend_task" {
  name   = "read-own-db-config"
  role   = aws_iam_role.backend_task.id
  policy = data.aws_iam_policy_document.backend_task.json
}

resource "aws_iam_role" "frontend_task" {
  name               = "dcta-frontend-task" # intentionally no policies
  assume_role_policy = data.aws_iam_policy_document.ecs_tasks_assume.json
}
```

## Validation & testing

Expected results for each check (illustrative shapes — replace with
captured output from a reference run before the cohort starts).

| Check | Expected |
|---|---|
| `nc -vz -w 5 <rds-host> 5432` from laptop | `Operation timed out` / no route |
| NAT Gateway count | `0` |
| `curl /` and `/api/tasks` | `200`; `3` |
| Backend at 0 | `503` on `/api/tasks`, `200` on `/` |
| Seed twice | Exit `0` twice; logs `INSERT 0 3` then `INSERT 0 0` |
| `sort rollout.log \| uniq -c` | e.g. `412 200` and nothing else |
| Broken image | `rolloutState: FAILED`, reason mentions circuit breaker; a later deployment `COMPLETED` on the previous revision |
| Blue/green test route | Test header returns the new `/health` `version`; plain route old, until production shift |
| Rollback | `describe-service-deployments` status `ROLLBACK_SUCCESSFUL`; rule 100 forwards to the original TG |
| Scaling | Activities "Successfully set desired count to 2/3", later back to 1; `describe-tasks` shows `FARGATE_SPOT` and `FARGATE` |
| IAM probe (backend) | `dcta/db/credentials`, `/dcta/db/host`, then 3 × `AccessDenied` |
| IAM probe (frontend) | 5 × `AccessDenied` |
| Teardown | `describe-load-balancers` → `0` |

Example probe log (backend role):

```text
--- expect ALLOW
dcta/db/credentials
/dcta/db/host
--- expect DENY
An error occurred (AccessDeniedException) when calling the GetSecretValue operation: User: arn:aws:sts::<ACCOUNT_ID>:assumed-role/dcta-backend-task/… is not authorized to perform: secretsmanager:GetSecretValue on resource: dcta/other-team/db …
An error occurred (AccessDeniedException) when calling the ListSecrets operation: …
An error occurred (AccessDenied) when calling the ListBuckets operation: …
```

## Output & usage notes

- ALB DNS name changes every session.
- Only the 3 seed rows persist conceptually; everything else is lost on destroy.
- The 70/30 split is only observable with ≥ 3 tasks.
- HTTP only (no domain, no certificate).

## Repository structure

Expected layout of a learner submission:

```text
<learner-project>/
├── app/                      # reference app + M1 changes (TLS, prod frontend, seed.sql)
├── infra/
│   ├── backend.hcl, env.sh   # state bucket + learner ID (no secrets)
│   ├── 10-foundation/        # vpc, security_groups, ecr, secrets, ci, outputs
│   ├── 20-data/              # RDS, subnet group, /dcta/db/host
│   └── 30-track-a/           # ecs, iam, alb, rules, services, seed, autoscaling, dashboard, bluegreen, task_roles, iam_probe
├── .gitlab-ci.yml            # local-phase jobs + build/deploy AWS jobs
└── docs/capstone/            # evidence for the Documentation Gate
```

## Documentation

The Documentation Gate evidence list is identical to the cohort version.
Check it is **real output** (timestamps, account IDs masked, SHAs that
match the commit history) and that the own-words explanation mentions:
foundation persists; data and track stacks are applied in dependency order
and destroyed in reverse; why (outputs consumed downstream; cost of
hourly resources).

## Checkpoint (self-assessed)

With expected answers:

- [ ] 1. VPC — `describe-subnets` shows 4 subnets, 2 per AZ; NAT count `0`.
- [ ] 2. SG chain — `dcta-rds-sg` ingress `UserIdGroupPairs` = `dcta-app-sg`, no `IpRanges`.
- [ ] 3. Stacks — three state keys under `dcta/`; `.terraform.lock.hcl` committed in each.
- [ ] 4. ECR — `imageTagMutability: IMMUTABLE`; tags are 7–8 char SHAs; image manifest `amd64`.
- [ ] 5. Secrets — `terraform state pull` contains no password value.
- [ ] 6. RDS — `PubliclyAccessible: false`; `nc` times out.
- [ ] 7. Task defs — `cpu 256`, `memory 512`, `X86_64`, secrets list of 4.
- [ ] 8. Spot — strategy `FARGATE_SPOT:7, FARGATE:3`; seed `INSERT 0 0` on rerun.
- [ ] 9. Routing — `503` / `200` in the backend-at-0 test.
- [ ] 10. CI — job log shows `register-task-definition` and `services-stable` succeeding.
- [ ] 11. Rollback — circuit-breaker `FAILED` deployment followed by the previous revision.
- [ ] 12. Blue/green — rollback status shown during bake time.
- [ ] 13. Logs — groups `/dcta/ecs/frontend|backend` with retention 3.
- [ ] 14. Scaling — activities to 3 and back to 1.
- [ ] 15. IAM — probe log as in **Validation & testing**.
- [ ] 16. Defense — within 30 minutes; `describe-load-balancers` → `0`.

## Project scope

Does: one app on ECS Fargate, single account/region, CI from gitlab.com,
blue/green + rolling, autoscaling, least privilege, daily rebuild/teardown.

Does not: Route 53/HTTPS, NAT/private-subnet compute, data persistence or
backups, multi-region, image scanning, app code changes beyond M1.

**Instructor notes — where the course differs from the original syllabus**
(do not share with learners):

- **No Trivy stage** in M4 (ADR-DCTA-AWS-E004-02 deprecated by stakeholder direction).
- **ECS-native blue/green instead of AWS CodeDeploy** in M6. CodeDeploy's ECS
  blue/green needs a dedicated listener per service and can't shift a path
  rule on a listener shared with the frontend; on the single-ALB design it
  would have redirected frontend traffic. Amends ADR-DCTA-AWS-E002-03 and PRD
  Epic 2 M6/AC 5 ("via CodeDeploy") — raise with the PRD owner.
- **CI identity is a GitLab OIDC role in `10-foundation`** (persistent), not
  an IAM user with an access key. ARCH §6 says "scoped IAM CI user"; the
  key would have been created outside Terraform (untracked) and would block
  the last-day destroy of the user. OIDC keeps every resource in Terraform
  and removes long-lived keys. It lives in the persistent stack so merges
  still build and push while `30-track-a` is destroyed.
- **Required app changes** (TLS for RDS PG 15+ `rds.force_ssl`, production
  frontend image, idempotent seed) are made once in M1.
- **No wrapper scripts**: daily `terraform apply` / `destroy` commands are
  explicit (ARCH §7.4's `up.sh`/`down.sh` are not used).
