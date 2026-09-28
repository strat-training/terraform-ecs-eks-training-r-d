# Reference: AWS Containers Bootcamp — Cited Sources

Raw research material for authoring `modules/shared-01-*` / `modules/track-[a,b]-*`
and `capstone/*`. Every URL below was fetched and returned HTTP 200 on
2026-09-28. Per `knowledge/patterns/module-content-structure.md`, a
module's **Supplemental Reading** may only cite URLs from this file (or
new URLs verified the same way and added here first).

Pinned versions resolved on 2026-09-28 via `resolve_package_versions`
(use these exact numbers in every snippet):

| Component | Version | Source of truth |
|---|---|---|
| Terraform CLI (min) | `>= 1.11` (S3 native lockfile needs 1.10; write-only `password_wo` / `secret_string_wo` need 1.11) | [S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3) |
| `hashicorp/aws` | `6.66.0` | resolve_package_versions |
| `hashicorp/helm` | `3.3.0` (v3 syntax: `kubernetes = {}` attribute, `set = [ {...} ]` list) | resolve_package_versions |
| `hashicorp/kubernetes` | `3.2.1` | resolve_package_versions |
| `hashicorp/random` | `3.9.1` | resolve_package_versions |
| `terraform-aws-modules/eks/aws` | `21.26.0` (v21 input names: `name`, `kubernetes_version`, `addons`, `endpoint_public_access`) | Terraform Registry API |
| EKS Kubernetes version | `1.36` (standard support to 2027-08-02) | [EKS versions](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html) |
| F5 NGINX Ingress Controller chart `nginx-ingress` | `2.7.3` (app `5.6.3`), repo `https://helm.nginx.com/stable` | resolve_package_versions |
| External Secrets Operator chart | `2.11.0`, repo `https://charts.external-secrets.io`, API `external-secrets.io/v1` | resolve_package_versions |
| Argo CD chart `argo-cd` | `10.9.2` (app `v3.5.3`), repo `https://argoproj.github.io/argo-helm` | resolve_package_versions |
| `kube-prometheus-stack` | `91.8.1`, repo `https://prometheus-community.github.io/helm-charts` | resolve_package_versions |
| `metrics-server` | `3.14.0` (app `0.9.0`), repo `https://kubernetes-sigs.github.io/metrics-server/` | resolve_package_versions |
| AWS CLI image (IAM probe) | `public.ecr.aws/aws-cli/aws-cli:2.37.4` | Docker Hub `amazon/aws-cli` tag API |
| Base images | `node:24-alpine`, `nginx:1.30-alpine`, `postgres:17-alpine`, `docker:29` / `docker:29-dind`, `alpine:3.24` (pulled via `public.ecr.aws/docker/library/…`) | Docker Hub tag API |

## Findings that changed the design (author-facing)

- **ingress-nginx retired March 2026** — no releases or security fixes after
  that date. Track B teaches F5 NGINX Ingress Controller instead (user
  decision 2026-09-28; amends ADR-DCTA-AWS-E003-06).
  [Retirement notice](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- **CodeDeploy ECS blue/green needs a dedicated listener per service** — it
  cannot shift a path rule (`/api/*`) on a shared listener. AWS: "CodeDeploy
  needs separate listeners for different services, and for production and
  test endpoints" ([AWS Containers blog, 2025-09-16](https://aws.amazon.com/blogs/containers/migrating-from-aws-codedeploy-to-amazon-ecs-for-blue-green-deployments)).
  M6 teaches ECS-native blue/green (`strategy = "BLUE_GREEN"`), which works
  at the listener-rule level (user decision 2026-09-28; amends
  ADR-DCTA-AWS-E002-03 and PRD Epic 2 M6/AC 5).
- **RDS for PostgreSQL 15+ defaults `rds.force_ssl = 1`** — the reference
  app's `pg` Pool has no SSL config and is refused.
  [RDS PostgreSQL SSL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Concepts.General.SSL.html)
- **ECS native blue/green exists (2025)** with bake time, lifecycle hooks and
  an ECS infrastructure role using `AmazonECSInfrastructureRolePolicyForLoadBalancers`.
- **EKS in-tree cloud provider** provisions a Classic LB for `type: LoadBalancer`
  unless annotated `service.beta.kubernetes.io/aws-load-balancer-type: nlb`.
- **Default VPC CNI limits `t3.medium` to 17 pods**; the full M15 stack needs ~20. Prefix delegation on the vpc-cni add-on (before node group creation) lifts managed nodes to the 110 cap.
- **F5 NIC rejects host-less Ingresses** unless `controller.allowEmptyIngressHost=true`, and only **one** host-less Ingress can own the default server (host collision, oldest wins). Grafana therefore uses a host rule (`grafana.dcta.test`) on the same NLB.
- **AWS now recommends EKS Pod Identity** for new workloads; IRSA remains
  supported and is what the PRD mandates.

## Sources by module

### M1 — VPC, ECR, RDS, Secrets, Terraform state
- [How Amazon VPC works](https://docs.aws.amazon.com/vpc/latest/userguide/how-it-works.html)
- [Security groups for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Security group rules (referencing other SGs)](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)
- [What is Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)
- [ECR lifecycle policies](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html)
- [Pushing a Docker image to ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html)
- [RDS DB instance in a VPC](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html)
- [Using SSL with a PostgreSQL DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Concepts.General.SSL.html)
- [Using SSL/TLS to encrypt a connection to a DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html)
- [Password management with RDS and Secrets Manager](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-secrets-manager.html)
- [What is AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [Terraform S3 backend (native lockfile)](https://developer.hashicorp.com/terraform/language/backend/s3)
- [terraform_remote_state data source](https://developer.hashicorp.com/terraform/language/state/remote-state-data)
- [aws_db_instance resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance)
- [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/)
- [Public IPv4 address charge announcement](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/)
- [Managing costs with AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [Docker multi-platform builds](https://docs.docker.com/build/building/multi-platform/)

### M2 — ECS, Fargate, Fargate Spot, ALB
- [What is Amazon ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html)
- [ECS task definitions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)
- [Task definition parameters (incl. `runtimePlatform`)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html)
- [Task execution IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)
- [Fargate capacity providers (FARGATE / FARGATE_SPOT)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html)
- [Fargate task networking (`awsvpc`)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html)
- [Fargate tasks and services (CPU/memory sizes)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-tasks-services.html)
- [Pass sensitive data to a container (`secrets`)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data-secrets.html)
- [What is an Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
- [ALB target groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)
- [AWS Fargate pricing](https://aws.amazon.com/fargate/pricing/)

### M3 — ALB listeners, rules, health checks
- [ALB listeners](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html)
- [Listener rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html)
- [Rule condition types (path-pattern, host-header, http-header)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-condition-types.html)
- [Target group health checks](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)

### M4 — GitLab CI, IAM for CI, ECS rolling deployments
- [GitLab CI/CD YAML reference](https://docs.gitlab.com/ci/yaml/)
- [GitLab CI/CD variables](https://docs.gitlab.com/ci/variables/)
- [Docker-in-Docker on GitLab](https://docs.gitlab.com/ci/docker/using_docker_build/)
- [GitLab OIDC to AWS](https://docs.gitlab.com/ci/cloud_services/aws/)
- [ECS rolling update deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html)
- [ECS deployment circuit breaker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html)
- [IAM security best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS CLI `ecs update-service`](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecs/update-service.html)

### M5 — CloudWatch logs, dashboards, autoscaling
- [Send ECS logs to CloudWatch (`awslogs`)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html)
- [CloudWatch dashboards](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Dashboards.html)
- [ECS service auto scaling — target tracking](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-autoscaling-targettracking.html)
- [Application Auto Scaling target tracking](https://docs.aws.amazon.com/autoscaling/application/userguide/application-auto-scaling-target-tracking.html)
- [CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)

### M6 — Blue/green on ECS
- [Amazon ECS blue/green deployments (native)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-blue-green.html)
- [ECS blue/green implementation (resources, roles)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/blue-green-deployment-implementation.html)
- [AmazonECSInfrastructureRolePolicyForLoadBalancers](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AmazonECSInfrastructureRolePolicyForLoadBalancers.html)
- [Migrating from CodeDeploy to ECS blue/green (AWS blog)](https://aws.amazon.com/blogs/containers/migrating-from-aws-codedeploy-to-amazon-ecs-for-blue-green-deployments)
- [CodeDeploy ECS deployments (legacy pattern)](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html)
- [CodeDeploy deployment configurations](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html)
- [AWS CLI `ecs stop-service-deployment`](https://docs.aws.amazon.com/cli/latest/reference/ecs/stop-service-deployment.html)
- [AWS CLI `ecs list-service-deployments`](https://docs.aws.amazon.com/cli/latest/reference/ecs/list-service-deployments.html)

### M7 — IAM task roles, least privilege
- [ECS task IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)
- [IAM JSON policy elements: Resource](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_resource.html)
- [Secrets Manager identity-based policy examples](https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access_examples.html)
- [Restricting access to Parameter Store parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-access.html)
- [Testing IAM policies with the policy simulator](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_testing-policies.html)
- [ECS Exec](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html)

### M9 — EKS cluster
- [Kubernetes cluster architecture](https://kubernetes.io/docs/concepts/architecture/)
- [What is Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [EKS managed node groups (incl. Spot)](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)
- [EKS Kubernetes version lifecycle](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
- [Connect kubectl to an EKS cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)
- [EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Choose an optimal EC2 node instance type (max pods)](https://docs.aws.amazon.com/eks/latest/userguide/choosing-instance-type.html)
- [Assign more IP addresses with prefixes (prefix delegation)](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)
- [EKS Best Practices — Prefix mode for Linux](https://docs.aws.amazon.com/eks/latest/best-practices/prefix-mode-linux.html)
- [terraform-aws-modules/eks/aws](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
- [Amazon EKS pricing](https://aws.amazon.com/eks/pricing/)
- [EKS Best Practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html)
- [EC2 Spot Instance interruptions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html)

### M10 — OIDC / IRSA
- [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Create an IAM OIDC provider for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)
- [Assign IAM roles to Kubernetes service accounts](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html)
- [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [Kubernetes service accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [EKS Best Practices — Identity and access management](https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html)

### M11 — External Secrets Operator
- [External Secrets Operator](https://external-secrets.io/latest/)
- [ESO AWS Secrets Manager provider](https://external-secrets.io/latest/provider/aws-secrets-manager/)
- [ClusterSecretStore API](https://external-secrets.io/latest/api/clustersecretstore/)
- [ExternalSecret API](https://external-secrets.io/latest/api/externalsecret/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Terraform `helm_release`](https://registry.terraform.io/providers/hashicorp/helm/latest/docs/resources/release)

### M12 — Helm on EKS
- [Helm values files](https://helm.sh/docs/chart_template_guide/values_files/)
- [Helm charts](https://helm.sh/docs/topics/charts/)
- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)

### M13 — Ingress
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [F5 NGINX Ingress Controller docs](https://docs.nginx.com/nginx-ingress-controller/)
- [Install NGINX Ingress Controller with Helm](https://docs.nginx.com/nginx-ingress-controller/installation/installing-nic/installation-with-helm/)
- [Ingress NGINX retirement notice](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [EKS network load balancing](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html)
- [F5 NIC host and listener collisions (`-allow-empty-ingress-host`)](https://docs.nginx.com/nginx-ingress-controller/configuration/host-and-listener-collisions/)
- [F5 NIC command-line arguments](https://docs.nginx.com/nginx-ingress-controller/configuration/global-configuration/command-line-arguments/)
- [Kubernetes Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)

### M14 — Argo CD / GitOps
- [Argo CD docs](https://argo-cd.readthedocs.io/en/stable/)
- [Argo CD core concepts](https://argo-cd.readthedocs.io/en/stable/core_concepts/)
- [Automated sync policy (prune, self-heal)](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [Declarative setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
- [GitLab deploy tokens](https://docs.gitlab.com/user/project/deploy_tokens/)
- [OpenGitOps principles](https://opengitops.dev/)

### M15 — Observability and HPA
- [kube-prometheus-stack chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [HPA walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [metrics-server](https://github.com/kubernetes-sigs/metrics-server)
- [Run Grafana behind a reverse proxy (sub-path)](https://grafana.com/tutorials/run-grafana-behind-a-proxy/)
- [EKS Best Practices — Cluster autoscaling](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-autoscaling.html)

### Capstones
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [Well-Architected Cost Optimization pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
