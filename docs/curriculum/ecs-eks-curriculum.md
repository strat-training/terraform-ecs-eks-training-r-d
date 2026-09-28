# 🎓 AWS Containers Bootcamp Syllabus (Intermediate Level)

## 📌 Curriculum Philosophy & Modular Architecture
- **Target Deployment Region:** `ap-southeast-1` (Singapore) — All provider configurations, pricing estimations, and latency baselines are pinned strictly to the Singapore region.
- **Unified Progressive Capstone App:** Every module builds directly toward or extends the primary **3-Tier Application Deployment** on AWS (React Web Front-end, Node.js App API, Amazon RDS PostgreSQL Database) with automated CI/CD and cloud observability.
- **Branching Track Design:** A common **Shared Infrastructure Setup** lays the network, registry, and database baseline. From there, learning path branches independently into **Track A: AWS ECS & Fargate** OR **Track B: AWS EKS**, depending on organizational or project requirements.
- **Guided vs. Unguided Paradigm:**
  - **Theory Sessions:** Self-paced text-based modules and architectural diagrams covering core concepts before hands-on implementation.
  - **Guided Activity:** Self-paced step-by-step activity workbook walking students through core concepts, AWS Management Console configurations, and foundational Terraform code.
  - **Unguided Capstone:** Students independently codify, automate, and extend the 3-tier architecture using Terraform and Kubernetes manifests without provided solutions.
- **FinOps Daily SOP (<$5 USD Budget Target):** At the start of active testing, students execute `terraform apply` / `eksctl create`. At the end of every lab session, students execute `terraform destroy` / `eksctl delete` to maintain a 100% sandbox-safe environment and guarantee total monthly account spend remains under $5.00 USD in `ap-southeast-1`.

---

## 🚫 Out of Scope (Explicitly Excluded to Protect Cost & Pacing)

To maintain a **strict < $5.00 USD budget per student**, preserve a **medium learning pace**, and avoid cognitive overload, the following topics are **explicitly excluded**:

1. **Amazon Route 53 & Custom Domain Names:** Avoids $0.50/month Hosted Zone fees and external domain purchase costs. All testing uses raw ALB and Ingress Load Balancer DNS URLs.
2. **AWS NAT Gateways:** Saves $32.85+/month per gateway. All EKS worker nodes and ECS workloads are intentionally deployed in Public Subnets with Public IPs to pull images from ECR for $0.00 in `ap-southeast-1`.
3. **AWS Cloud Map & Service Discovery:** Replaced by native ALB Path-Based Listener Rules and Kubernetes Ingress routing to keep networking concepts streamlined.
4. **Multi-Region / Multi-Cloud Deployments:** All labs are pinned strictly to `ap-southeast-1` (Singapore).
5. **Stateful Container Storage (PersistentVolumes / EBS CSI Drivers):** Relational databases run exclusively on managed Amazon RDS (`db.t4g.micro` Free Tier) to teach stateful/stateless decoupling rather than running databases inside containers.

---

## 🏗️ Phase 1: Shared Infrastructure Setup (Common Foundation)

### Module 1: AWS VPC, ECR & RDS Database Setup (`ap-southeast-1`)
* **Core Concepts:** Terraform state, VPC subnet topology in Singapore (`ap-southeast-1a`/`ap-southeast-1b`), Security Group chaining, Amazon ECR, Amazon RDS PostgreSQL (Free Tier).
* **Theory Sessions:**
  - *Session 1.1: AWS Global Infrastructure & VPC Networking 101 (CIDR, Subnets, Security Groups).*
  - *Session 1.2: Container Registries and Amazon ECR.*
  - *Session 1.3: Stateless Compute vs. Stateful Managed Storage (Amazon RDS).*
* **Guided Activity:**
  1. Review AWS VPC architecture across `ap-southeast-1` Availability Zones, public/private subnets, and security group chaining concepts.
  2. Follow the guided workbook to provision an Amazon ECR repository in `ap-southeast-1` and push the React and Node.js Docker images using the AWS CLI.
  3. Inspect the provided sample Terraform module for an `aws_db_instance` (PostgreSQL `db.t4g.micro`).
* **Unguided Capstone Challenge:**
  * Write Terraform code to provision a custom VPC across 2 Availability Zones in `ap-southeast-1` (2 Public, 2 Private subnets) and an RDS PostgreSQL instance in the private subnets. Store the database credentials in **AWS Secrets Manager** and enforce Security Group chaining so port `5432` accepts inbound traffic strictly from the application tier.

---

## 🚀 Track A: AWS ECS & Fargate Track (AWS-Native Orchestration)

### Module 2: Low-Cost ECS Fargate Compute & Load Balancing
* **Core Concepts:** ECS Clusters, Task Definitions, Low-Cost Fargate Spot Capacity Providers, Application Load Balancers (ALB), Target Groups in `ap-southeast-1`.
* **Theory Sessions:**
  - *Session 2.1: Container Orchestration Basics & AWS ECS Architecture.*
  - *Session 2.2: Serverless Compute with AWS Fargate & Fargate Spot Cost Rules.*
  - *Session 2.3: Layer 7 Load Balancing with AWS Application Load Balancers.*
* **Guided Activity:**
  1. Explore the ECS Console in `ap-southeast-1` to understand Task Definitions, Task Execution Roles, and Fargate networking (`awsvpc` mode).
  2. Follow the workbook to configure an ALB with listener rules and target groups routing traffic on port `80`.
* **Unguided Capstone Challenge:**
  * Write Terraform code to define ECS Task Definitions for both React and Node.js containers. Apply the **FinOps Low-Cost Rule** directly in Terraform by setting the ECS Service capacity provider strategy to 70% **Fargate Spot** and 30% Fargate On-Demand. Configure the backend task definition to fetch database credentials dynamically from AWS Secrets Manager using the `secrets` block and deploy behind the ALB.

---

### Module 3: Advanced ALB Path-Based Routing & Listener Rules
* **Core Concepts:** ALB Listener Rules, Path-Based Routing (`/api` vs `/`), Host-Based Routing, ALB Health Checks (No Route 53 needed).
* **Theory Sessions:**
  - *Session 3.1: Advanced HTTP Routing (Path-Based vs Host-Based).*
  - *Session 3.2: Application Layer Traffic Management and Health Checks.*
* **Guided Activity:**
  1. Learn how a single Application Load Balancer in `ap-southeast-1` routes traffic to multiple Target Groups based on URL paths.
  2. Walk through configuring listener rules in the AWS Console.
* **Unguided Capstone Challenge:**
  * Write Terraform code to update the Web ALB listener rules so that requests to `/api/*` are routed to the Node.js backend Target Group, while all default requests (`/*`) route to the React frontend Target Group—all served under a single raw ALB DNS URL in `ap-southeast-1`.

---

### Module 4: GitLab CI/CD & Automated ECS Rollouts
* **Core Concepts:** Programmatic IAM permissions, GitLab CI variables, container vulnerability scanning (Trivy), zero-downtime rolling updates in `ap-southeast-1`.
* **Theory Sessions:**
  - *Session 4.1: CI/CD Principles for Containerized Microservices.*
  - *Session 4.2: Securely Authenticating CI Pipelines with AWS IAM.*
  - *Session 4.3: Deployment Strategies: Rolling Updates vs. Recreate.*
* **Guided Activity:**
  1. Set up an IAM User with scoped permissions for ECR and ECS updates in `ap-southeast-1`, and store the keys in GitLab CI/CD variables.
  2. Walk through a sample `.gitlab-ci.yml` file configuring Docker build and push stages to ECR in `ap-southeast-1`.
* **Unguided Capstone Challenge:**
  * Update `.gitlab-ci.yml` to include a container vulnerability scanning stage using Trivy and an automated deployment step using `aws ecs update-service --region ap-southeast-1`. Prove that merging a code change to `main` automatically triggers a zero-downtime rolling update on ECS Fargate.

---

### Module 5: CloudWatch Observability & Auto-Scaling
* **Core Concepts:** `awslogs` driver, CloudWatch Log Groups, CloudWatch Dashboards, ECS Target Tracking Auto-Scaling in `ap-southeast-1`.
* **Theory Sessions:**
  - *Session 5.1: Cloud Observability 101: Metrics, Logs, and Traces.*
  - *Session 5.2: Routing Container Logs with the awslogs Driver.*
  - *Session 5.3: Application Auto-Scaling Mechanisms in AWS.*
* **Guided Activity:**
  1. Review log drivers in ECS Task Definitions and search log streams in the CloudWatch Logs Console (`ap-southeast-1`).
  2. Walk through Application Auto Scaling concepts for ECS Fargate services.
* **Unguided Capstone Challenge:**
  * Update the Terraform Task Definitions to route all stdout/stderr logs to `aws_cloudwatch_log_group` resources in `ap-southeast-1`. Write Terraform code using `aws_appautoscaling_target` and `aws_appautoscaling_policy` to automatically scale the backend Fargate service from 1 to 3 tasks when average CPU utilization exceeds 70%.

---

### Module 6: AWS CodeDeploy for Blue/Green ECS Rollouts
* **Core Concepts:** AWS CodeDeploy in `ap-southeast-1`, Blue/Green deployment controller, ALB test listeners, automated rollback triggers.
* **Theory Sessions:**
  - *Session 6.1: Advanced Deployment Strategies (Blue/Green vs Canary).*
  - *Session 6.2: AWS CodeDeploy Architecture for ECS Fargate.*
* **Guided Activity:**
  1. Walk through the AWS CodeDeploy console in `ap-southeast-1` and inspect the `appspec.yaml` syntax required for ECS tasks.
  2. Review the mechanics of shifting traffic between primary and test target groups on an ALB.
* **Unguided Capstone Challenge:**
  * Update the ECS Service deployment configuration in Terraform to use `CODE_DEPLOY` instead of standard `ECS` rolling updates. Configure a dual-target-group setup on the ALB and execute a Blue/Green deployment using the AWS CLI or GitLab CI.

---

### Module 7: IAM Task Roles & Granular Access Control
* **Core Concepts:** Task Execution Roles vs. App Task Roles, IAM Least Privilege, AWS Systems Manager (SSM) Parameter Store in `ap-southeast-1`.
* **Theory Sessions:**
  - *Session 7.1: Identity and Access Management (IAM) Least Privilege.*
  - *Session 7.2: Decoupling ECS Task Roles from Task Execution Roles.*
  - *Session 7.3: Secrets Management Lifecycles (SSM vs Secrets Manager).*
* **Guided Activity:**
  1. Learn how to separate AWS API permissions needed by the ECS agent (pulling images) from permissions needed by application code (reading parameters).
* **Unguided Capstone Challenge:**
  * Refactor the ECS Task Roles in Terraform so that the Node.js container is granted read access *only* to its specific database secret ARN in AWS Secrets Manager and Parameter Store in `ap-southeast-1`, blocking all other AWS API calls.

---

### Module 8: Track A Capstone Integration & ECS Defense
* **Core Concepts:** Production readiness, pipeline verification, Cloud FinOps, live architectural defense.
* **Theory Sessions:**
  - *Session 8.1: Architecting ECS for Production (Review).*
  - *Session 8.2: Technical Defense Preparation & Presentation Skills.*
* **Guided Activity:**
  1. Review the ECS self-audit checklist ensuring all ECS services in `ap-southeast-1`, Blue/Green deployments, CloudWatch dashboards, and CI/CD triggers are fully functional.
* **Unguided Capstone Challenge:**
  * Conduct a live 15-minute ECS Capstone Defense: trigger a git commit to demonstrate an automated Blue/Green rollout on ECS Fargate, showcase CloudWatch auto-scaling, and defend the Terraform modular architecture. Execute `terraform destroy` as part of the daily FinOps SOP.

---

## ☸️ Track B: AWS EKS Track (Enterprise Cloud Kubernetes)

### Module 9: Low-Cost EKS Cluster Provisioning (Terraform)
* **Core Concepts:** `terraform-aws-modules/eks/aws`, EKS Control Plane in `ap-southeast-1`, Managed Node Groups, Single EC2 Spot Instance (`t3.small` / `t4g.small`).
* **Theory Sessions:**
  - *Session 9.1: Kubernetes Architecture (Control Plane vs. Worker Nodes).*
  - *Session 9.2: Amazon EKS Under the Hood.*
  - *Session 9.3: Low-Cost Node Provisioning Rules (EC2 Spot in Public Subnets).*
* **Guided Activity:**
  1. Review the architecture of the AWS-managed Kubernetes control plane and node group provisioning models in `ap-southeast-1`.
  2. Follow the workbook to configure local CLI tools (`aws eks update-kubeconfig --region ap-southeast-1`) and verify cluster connectivity.
* **Unguided Capstone Challenge:**
  * Write Terraform code to provision an EKS cluster in `ap-southeast-1` public subnets using a **single EC2 Spot Instance** (`t3.small` / `t4g.small`) to eliminate NAT Gateway fees ($0.00 NAT). Integrate the cluster into the VPC created in Module 1 so worker nodes can reach the RDS PostgreSQL instance.

---

### Module 10: OIDC Federation & IAM Roles for Service Accounts (IRSA)
* **Core Concepts:** OpenID Connect (OIDC) Issuer, IAM Roles for Service Accounts (IRSA), fine-grained pod security in `ap-southeast-1`.
* **Theory Sessions:**
  - *Session 10.1: Identity Federation in the Cloud (OIDC Deep Dive).*
  - *Session 10.2: Mapping Kubernetes Service Accounts to AWS IAM (IRSA).*
* **Guided Activity:**
  1. Walk through the mechanics of how Kubernetes Service Accounts assume AWS IAM Roles via OIDC tokens.
* **Unguided Capstone Challenge:**
  * Write Terraform code to enable the EKS OIDC provider in `ap-southeast-1` and create an IAM Role granting read access to AWS Secrets Manager. Annotate a Kubernetes Service Account with the IAM Role ARN and verify that a test Pod can assume the role.

---

### Module 11: External Secrets Operator (ESO) & Dynamic Syncing
* **Core Concepts:** External Secrets Operator, `ClusterSecretStore`, `ExternalSecret` Custom Resources referencing `ap-southeast-1`.
* **Theory Sessions:**
  - *Session 11.1: Kubernetes Secrets Management Pitfalls.*
  - *Session 11.2: The Kubernetes Operator Pattern & External Secrets Operator.*
* **Guided Activity:**
  1. Learn how ESO bridges external cloud secret stores into native Kubernetes `Secret` objects.
  2. Follow the workbook to deploy ESO into the cluster using the Terraform Helm Provider (`hashicorp/helm`).
* **Unguided Capstone Challenge:**
  * Write the `ClusterSecretStore` (configured for region `ap-southeast-1`) and `ExternalSecret` manifests to automatically sync the RDS PostgreSQL credentials from AWS Secrets Manager into a native Kubernetes `Secret` within the application namespace.

---

### Module 12: Local Helm Chart Migration & Cloud Deployment
* **Core Concepts:** Helm values parameterization, ConfigMaps, Secrets injection, `kubectl` troubleshooting.
* **Theory Sessions:**
  - *Session 12.1: Package Management with Helm.*
  - *Session 12.2: Configuration Injection (ConfigMaps & Secrets).*
  - *Session 12.3: Cloud Kubernetes Troubleshooting Methodologies.*
* **Guided Activity:**
  1. Review the 3-tier application Helm charts built during the local Minikube module.
* **Unguided Capstone Challenge:**
  * Refactor the local Helm charts to consume the database credentials generated by ESO and deploy the React frontend and Node.js backend onto the EKS cluster in `ap-southeast-1`. Verify backend communication with the cloud RDS instance.

---

### Module 13: Cloud Ingress Controllers (NGINX Ingress)
* **Core Concepts:** Ingress Controllers, NGINX Ingress, AWS Classic/Network Load Balancer provisioning in `ap-southeast-1` (No Route 53 needed).
* **Theory Sessions:**
  - *Session 13.1: Kubernetes Ingress vs AWS ALBs.*
  - *Session 13.2: NGINX Ingress Controller Architecture.*
  - *Session 13.3: External Traffic Flow into EKS.*
* **Guided Activity:**
  1. Review Kubernetes `Ingress` resource spec and load balancer controller integrations on AWS.
* **Unguided Capstone Challenge:**
  * Use the Terraform Helm Provider to deploy the NGINX Ingress Controller to EKS. Update the React frontend Helm chart to include an `Ingress` resource and verify that AWS automatically provisions a Load Balancer in `ap-southeast-1` routing public HTTP traffic to the pods via the raw Load Balancer URL.

---

### Module 14: Cloud GitOps Workflow with ArgoCD
* **Core Concepts:** ArgoCD, Declarative Applications, GitOps sync policies, automated drift correction.
* **Theory Sessions:**
  - *Session 14.1: The GitOps Philosophy (Pull vs Push CI/CD).*
  - *Session 14.2: ArgoCD Architecture and Sync States.*
* **Guided Activity:**
  1. Explore the ArgoCD web UI and review declarative `Application` manifest structures.
* **Unguided Capstone Challenge:**
  * Deploy ArgoCD into the EKS cluster using Terraform. Connect ArgoCD to the team's GitLab repository and create an ArgoCD Application manifest that automatically syncs and deploys the 3-tier application Helm charts.

---

### Module 15: Cloud Observability & Horizontal Pod Autoscaling (HPA)
* **Core Concepts:** Prometheus Operator, Grafana, Kubernetes Metrics Server, Horizontal Pod Autoscaler (HPA).
* **Theory Sessions:**
  - *Session 15.1: Monitoring Kubernetes with Prometheus and Grafana.*
  - *Session 15.2: Autoscaling Patterns: HPA vs. VPA vs. Cluster Autoscaler.*
* **Guided Activity:**
  1. Learn how the Metrics Server collects pod metrics and triggers HPA scaling events.
* **Unguided Capstone Challenge:**
  * Deploy the Prometheus/Grafana stack using ArgoCD and expose Grafana via an Ingress route. Write an `HorizontalPodAutoscaler` manifest for the Node.js backend targeting 70% CPU. Run a load test against the public URL in `ap-southeast-1` and capture Grafana metrics showing HPA scaling events.

---

### Module 16: Track B Capstone Integration & EKS Defense
* **Core Concepts:** Production readiness, architectural defense, complete environment teardown.
* **Theory Sessions:**
  - *Session 16.1: Production Kubernetes Architectures (Review).*
  - *Session 16.2: Final FinOps Teardown Procedures.*
* **Guided Activity:**
  1. Prepare the EKS capstone demonstration and review the evaluation criteria.
* **Unguided Capstone Challenge:**
  * Conduct the final live defense: trigger a git commit to demonstrate an automated ArgoCD sync on EKS, present a structured decision matrix defending your EKS build experience, and execute `eksctl delete cluster --region ap-southeast-1` and `terraform destroy` to completely wipe all AWS resources.