# Track B Capstone Project: Task Manager on AWS EKS with GitOps

## Project Overview

This capstone project takes the task-manager app, Helm chart and GitOps
workflow you built on Minikube and runs them on **Amazon EKS**. You'll
build the cluster with **Terraform**, sync secrets from AWS with the
**External Secrets Operator**, expose the app through **NGINX Ingress**,
deploy everything from Git with **Argo CD**, and watch it scale with
**Prometheus, Grafana and a Horizontal Pod Autoscaler**. You'll show that
you can run Kubernetes in the cloud — and rebuild the whole cluster from
your repository every day.

## Provided Application

You'll keep working with the same task-manager application from the local
phase:

- React frontend
- Node.js/Express backend
- PostgreSQL database (now on **Amazon RDS**)

The code is already working. In M1 you made three small changes so it
runs on AWS — TLS for the database connection, a production frontend
image, and a re-runnable `seed.sql`. Your `task-app` Helm chart from the
local phase is your starting point for the cluster side.

## Learning Objectives

By completing this project, you will:

1. Build AWS infrastructure with Terraform, split into stacks you can
   create and destroy safely every day
2. Provision an EKS cluster with a single low-cost Spot node
3. Give pods AWS permissions with IAM Roles for Service Accounts (IRSA)
4. Sync secrets from AWS into Kubernetes without putting them in Git
5. Adapt one Helm chart for both Minikube and EKS
6. Expose apps through one load balancer with an Ingress controller
7. Run a full GitOps workflow with Argo CD, including self-heal
8. Monitor the cluster with Prometheus and Grafana
9. Scale pods automatically with a Horizontal Pod Autoscaler

## Project Requirements

Each area below matches the modules you complete. The full list of
acceptance criteria is in your capstone spec,
`capstone-track-b-eks-gitops-cohort.md`.

### 1. AWS Foundation (M1)
- Custom VPC in `ap-southeast-1` across 2 Availability Zones
  * 2 public and 2 private subnets
  * Internet Gateway, no NAT Gateway
- Security groups chained: app tier → database
  * Internet traffic only from your own IP (`/32`)
  * Database port `5432` open only to the app tier
- Amazon ECR repositories for the frontend and backend
  * Images built for `linux/amd64` and tagged with the Git commit
- Amazon RDS PostgreSQL in the private subnets
  * Credentials in AWS Secrets Manager, never in Git or Terraform state
- Terraform remote state in S3 with locking

### 2. EKS Cluster & Workload Identity (M9–M10)
- EKS cluster built with Terraform
  * API endpoint reachable only from your IP
  * One Spot worker node in a public subnet
  * Enough pod capacity for the whole stack (prefix delegation)
- Pods can reach RDS through the security-group chain
- IRSA role for the External Secrets Operator
  * Usable by one ServiceAccount only
  * Allowed to read only the DB secret and parameters

### 3. Secrets & Helm Deployment (M11–M12)
- External Secrets Operator installed with Terraform
- Kubernetes Secrets created from AWS Secrets Manager and Parameter Store
  * No password anywhere in Git
- One `task-app` Helm chart with an EKS values file
  * Images from ECR, no in-cluster database
  * Resource requests, limits and health probes
  * A seed Job that loads the database

### 4. Ingress & GitOps (M13–M14)
- NGINX Ingress Controller installed with Terraform
  * One Network Load Balancer, open only to your IP
- `Ingress` routing `/api` to the backend and `/` to the frontend
- Argo CD installed with Terraform, UI kept private
- Argo CD Applications for your platform, secrets and app
  * Automated sync, prune and self-heal
  * Private repo access with a read-only deploy token

### 5. Observability & Autoscaling (M15)
- metrics-server and the Prometheus/Grafana stack deployed through Argo CD
- Grafana reachable on the same load balancer
- HorizontalPodAutoscaler for the backend: 1 to 3 pods at 70% CPU
- Load test with scaling visible in Grafana

## Getting Started

1. Make sure your local-phase gitlab.com project, Helm chart and Argo CD
   setup still work on Minikube
2. Get your AWS sandbox credentials and Terraform state bucket name from
   the program
3. Start with the M1 content pack and build the foundation
4. Follow the daily routine every session (full commands, including `init`
   and your settings, are in the content packs):
```bash
# Start of session: apply in order, then hand the cluster to Argo CD
terraform -chdir=infra/10-foundation apply
terraform -chdir=infra/20-data apply
terraform -chdir=infra/40-track-b-cluster apply
terraform -chdir=infra/41-track-b-platform apply
kubectl apply -f deploy/argocd/

# End of session: destroy in reverse order, checking the load balancer is gone first
terraform -chdir=infra/41-track-b-platform destroy

# Before removing the cluster, check the load balancer is really gone (should print 0;
# if not, wait a minute and check again — never destroy the cluster while it's there)
aws elbv2 describe-load-balancers --query 'length(LoadBalancers)'

terraform -chdir=infra/40-track-b-cluster destroy
terraform -chdir=infra/20-data destroy
```
5. Keep your live cluster time to about 3 hours a day
6. Save your evidence in `docs/capstone/` as you go — don't leave it for
   the last day

## Two-Week Timeline

### Week 5: Foundation, Cluster and Secrets
- Days 1-2: AWS Foundation (M1)
  * Set up Terraform remote state and your stacks
  * Build the VPC, security groups, ECR and the DB secret
  * Push your frontend and backend images to ECR
  * Create and destroy RDS with Terraform

- Day 3: EKS Cluster (M9)
  * Write the EKS Terraform with one Spot node
  * Connect with `kubectl`
  * Check a pod can reach RDS

- Day 4: IRSA and External Secrets (M10–M11)
  * Create the IRSA role and test it with pods
  * Install External Secrets with Terraform
  * Sync the DB settings into Kubernetes Secrets

- Day 5: Helm on EKS (M12)
  * Create your EKS values file
  * Add the seed Job
  * Deploy the app and test it through port-forward

### Week 6: Traffic, GitOps and Operations
- Day 1: NGINX Ingress (M13)
  * Install the ingress controller with Terraform
  * Add the Ingress to your chart
  * Open the app in a browser through the load balancer

- Day 2: GitOps with Argo CD (M14)
  * Install Argo CD with Terraform
  * Connect your GitLab repo with a deploy token
  * Create your Applications and test self-heal

- Day 3: Observability and Autoscaling (M15)
  * Deploy metrics-server and the monitoring stack with Argo CD
  * Add the HPA to your chart
  * Load test and capture Grafana graphs

- Day 4: Integration and Documentation
  * Rebuild the whole environment from scratch and check it all syncs
  * Write your decision matrix
  * Finish your capstone documentation

- Day 5: Capstone Defense (M16)
  * Run through the 15-minute demo
  * Live defense and Q&A
  * Final `terraform destroy` and teardown check

## Evaluation Criteria

Your capstone is graded with the **Capstone Grading Rubric**
(`capstone-grading-rubric.md`):

| Phase | Weight |
|---|---|
| Technical Execution | 45% |
| Functional Demonstration | 35% |
| Presentation & Defense | 20% |

Plus the **Technical Documentation Gate** (pass/fail): your evidence in
`docs/capstone/` must be complete and real. If it isn't, the capstone
doesn't pass, whatever the score.

### Not Yet Passing (below 75%)
- Missing or broken requirements
- Passwords or tokens in Git, or open security groups
- Can't explain how the system works

### Proficient Implementation (75-89%)
- All requirements working in the live demo
- Terraform stacks clean, versions pinned, secrets out of Git and state
- IRSA and External Secrets working, with denied-call proof
- Argo CD syncing and self-healing everything in the cluster
- Monitoring and HPA working under load
- Clear explanations of your choices and a complete decision matrix

### Advanced Implementation (90-100%)
- Everything in Proficient, plus:
- Handles the live destructive tests calmly and correctly
- Explains trade-offs between options with confidence
- Well-organized, easy-to-extend Terraform, charts and manifests
- One or more stretch goals below
- Complete, clear documentation

## Stretch Goals

These don't add extra points on their own, but they're the kind of work
that earns **Exemplary** on "Application of Trained Skills":

1. **Security**
   - EKS Pod Identity instead of IRSA for a second workload
   - Kubernetes NetworkPolicies limiting which pods can talk to the backend

2. **Advanced Features**
   - An "app of apps" root Application that creates all the others
   - A GitLab CI job that updates the image tag in `values-eks.yaml` after a build

3. **Advanced Observability**
   - A custom Grafana dashboard for the task-manager app
   - `ServiceMonitor` scraping app metrics you instrumented in the local phase

## Deliverables

1. **GitLab Repository**
   - Application code (with the M1 changes)
   - Terraform stacks under `infra/`
   - Helm chart, ESO manifests and Argo CD Applications under `deploy/`
   - Documentation

2. **Documentation** (in `docs/capstone/`)
   - `terraform apply` / `terraform destroy` output for each stack
   - Proof the database is private and the node's pod capacity
   - IRSA allowed and denied test results
   - External Secrets status and proof no password is in Git
   - Routing evidence through the load balancer
   - Argo CD sync, self-heal and revert evidence
   - HPA events and a Grafana screenshot
   - Teardown proof
   - `decision-matrix.md`
   - Your daily apply/destroy order, explained in your own words

3. **Presentation** (15 minutes, live)
   - Git commit triggering an Argo CD sync
   - Self-heal after a manual change
   - HPA scaling under load in Grafana
   - Walkthrough of your decision matrix
   - Clean teardown

## Support Resources

### Training Materials
Your content packs for this track:
1. Week 5 — AWS Shared Foundation: VPC, ECR, RDS & Secrets (M1)
2. Week 5 — Track B: Low-Cost EKS Cluster & IAM Roles for Service Accounts (M9–M10)
3. Week 5 — Track B: External Secrets Operator & Helm on EKS (M11–M12)
4. Week 6 — Track B: NGINX Ingress & GitOps with Argo CD (M13–M14)
5. Week 6 — Track B: Cloud Observability & Horizontal Pod Autoscaling (M15)
6. Capstone spec: `capstone-track-b-eks-gitops-cohort.md`
7. Capstone Grading Rubric: `capstone-grading-rubric.md`

### Documentation
- Official documentation (for additional reference):
  * [Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)
  * [What is Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
  * [terraform-aws-modules/eks/aws](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
  * [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
  * [External Secrets Operator](https://external-secrets.io/latest/)
  * [Helm values files](https://helm.sh/docs/chart_template_guide/values_files/)
  * [F5 NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/)
  * [Argo CD](https://argo-cd.readthedocs.io/en/stable/)
  * [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
  * [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

Remember:
- Your Helm chart and GitOps skills from the local phase carry straight over
- Apply your stacks at the start of every session and destroy them at the end
- Destroy the platform stack before the cluster, every time
- Make changes in Git — Argo CD will undo manual ones
- Never put a password or token in Git
- Save your evidence as you go, not on the last day
- Ask questions early during lab sessions
