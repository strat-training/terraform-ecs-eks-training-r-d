# Capstone — Track B: Task Manager on EKS with GitOps (Instructor Version)

In this capstone the cohort brings together M1 and M9–M15: the task-manager
app on **Amazon EKS**, secrets synced by **External Secrets Operator**,
traffic through **one F5 NGINX Ingress NLB**, everything deployed from Git
by **Argo CD**, observed with **Prometheus/Grafana**, scaled by an **HPA** —
all in **Terraform**, rebuilt every session.

> **Disclaimer**
>
> The TaskFlow team is fictional. Built for the DevOps Bootcamp's
> *Containerization in AWS* phase (DCTA-AWS), using the reference app
> `devops-capstone-3tier-app`.

> **Instructor-only content.** This file contains the reference solution,
> expected outputs and checkpoint answers. Give learners
> `capstone-track-b-eks-gitops-cohort.md` instead.

## Getting started

### Prerequisites

Same as the cohort version: sandbox account in `ap-southeast-1`, state
bucket, learner's gitlab.com project with M1 app changes and
`deploy/charts/task-app`, a read-only deploy token, and Terraform 1.11+,
AWS CLI v2, Docker `buildx`, `kubectl`, `helm` 3, `eksctl`, `git`, `curl`,
`jq`, `dig`, `ab`.

### Local setup

Walk a reference build while validating a submission. From the repo root:

```bash
# Settings and today's IP
source infra/env.sh
export TF_VAR_learner_cidr="$(curl -fsS https://checkip.amazonaws.com)/32"

# Apply in dependency order
terraform -chdir=infra/10-foundation init -backend-config=../backend.hcl
terraform -chdir=infra/10-foundation apply
terraform -chdir=infra/20-data init -backend-config=../backend.hcl
terraform -chdir=infra/20-data apply
terraform -chdir=infra/40-track-b-cluster init -backend-config=../backend.hcl
terraform -chdir=infra/40-track-b-cluster apply
aws eks update-kubeconfig --region ap-southeast-1 --name dcta-eks
terraform -chdir=infra/41-track-b-platform init -backend-config=../backend.hcl
terraform -chdir=infra/41-track-b-platform apply

# Session-only secrets, created from the shell
kubectl -n argocd create secret generic gitlab-repo \
  --from-literal=type=git \
  --from-literal=url="https://gitlab.com/<group>/<project>.git" \
  --from-literal=username="<deploy-token-username>" \
  --from-literal=password="$GITLAB_DEPLOY_TOKEN"
kubectl -n argocd label secret gitlab-repo argocd.argoproj.io/secret-type=repository
kubectl create namespace monitoring
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 18)"

# Bootstrap everything else from Git
kubectl apply -f deploy/argocd/
```

Verification — one happy path, one failure path:

```bash
LB="$(kubectl -n nginx-ingress get svc -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')"

# Happy path
kubectl -n argocd get applications
curl -s -o /dev/null -w '%{http_code}\n' "http://$LB/"
curl -s "http://$LB/api/tasks" | jq length

# Failure path: self-heal
kubectl -n task-app delete deploy backend
kubectl -n task-app get deploy backend -w
```

Teardown:

```bash
terraform -chdir=infra/41-track-b-platform destroy

# Before removing the cluster, check the load balancer is really gone (should print 0;
# if not, wait a minute and check again — never destroy the cluster while it's there)
aws elbv2 describe-load-balancers --query 'length(LoadBalancers)'

terraform -chdir=infra/40-track-b-cluster destroy
terraform -chdir=infra/20-data destroy
```

### Data setup

No external data: `seed.sql` (from the reference app's `init.sql`, M1) is
copied into the chart's `files/` and run by the `db-seed` hook Job on every
sync.

## Business problem

> Can a small team run its task-manager app on Kubernetes in AWS so that
> Git is the only way to change what's running, no password ever lives in
> Git, the app scales by itself, and the whole cluster can be rebuilt from
> the repository in minutes?

## Requirements / acceptance criteria

Each requirement is followed by **how the reference solution satisfies
it**. Snippets are in **Key engineering features**.

1–6. **Foundation (M1).** *Solution:* identical to the Track A instructor
     version, snippet **A-1** (`vpc.tf`, `security_groups.tf`, `outputs.tf`,
     `20-data`). The public subnets' `map_public_ip_on_launch = true` and
     `kubernetes.io/role/elb = 1` tag matter for Track B (node public IP;
     load balancer subnet discovery). The `app_self` ingress rule covers
     node ↔ pod traffic.
7. **EKS 1.36 via module 21.26.0, `/32` endpoint, no control-plane log group (M9).** *Solution:* **B-1** (`enabled_log_types = []`, `create_cloudwatch_log_group = false`).
8. **One Spot node, `dcta-app-sg`, prefix delegation, pod → RDS (M9).**
   *Solution:* **B-1** — `vpc-cni` add-on with `before_compute = true` and
   `ENABLE_PREFIX_DELEGATION`; node group `vpc_security_group_ids`.
9. **IRSA pinned and scoped; denied tests (M10).** *Solution:* **B-2**.
10. **ESO via `helm_release` with IRSA (M11).** *Solution:* given in the M11 lab.
11. **Stores + ExternalSecrets; forbidden-secret failure (M11).** *Solution:* **B-3**.
12. **One chart + `values-eks.yaml`; bad tag diagnosed (M12).** *Solution:* **B-4**.
13. **NGINX Ingress, NLB, `/32`, host-less, `wait` (M13).** *Solution:* **B-5**.
14. **Ingress routing; backend-at-0 failure (M13).** *Solution:* **B-4** (`ingress.yaml`).
15. **Argo CD via Terraform, private UI, token from shell (M14).** *Solution:* **B-5**.
16. **Applications with auto sync/prune/self-heal (M14).** *Solution:* **B-6**.
17. **metrics-server + kube-prometheus-stack via Argo CD; Grafana host (M15).**
    *Solution:* M15 lab (metrics-server) and **B-7**.
18. **HPA 1 → 3 at 70%; missing-request failure (M15).** *Solution:* **B-4**
    (`hpa.yaml`, Deployment without `replicas` when autoscaling is on).
19. **Defense with decision matrix (M16).** *Solution:* see the example
    matrix in **Deliverable**.
20. **Teardown in reverse order (M16).** *Solution:* `41` → `40` → `20`;
    `helm_release.nginx_ingress` has `wait = true`, `timeout = 900`, so the
    NLB is gone before the cluster is destroyed.

## Deliverable

Running system + evidence + `docs/capstone/decision-matrix.md`.

Example of a strong decision matrix (for calibration; learners must write
their own from their build):

| Decision | What I chose | Alternative I considered | Trade-off | Evidence from my build |
|---|---|---|---|---|
| Pod AWS credentials | IRSA pinned to one ServiceAccount | Static keys in a Secret; EKS Pod Identity | No long-lived keys; needs OIDC provider + exact `sub` | Allowed vs. `default` SA test-pod logs |
| DB password delivery | ESO from Secrets Manager | Sealed/committed Secret; Helm value | Nothing sensitive in Git; extra operator to run | `SecretSynced`; `git grep` empty |
| Traffic entry | One NGINX Ingress controller behind one NLB | `LoadBalancer` Service per app; AWS LB Controller + ALB | One LB, simple; only one host-less Ingress allowed | `describe-load-balancers` shows 1 NLB |
| Deployment model | Argo CD pull, self-heal | CI `helm upgrade` push | No cluster creds in CI; drift reverted | Commit → `Synced`; deleted Deployment restored |
| Node sizing | 1 × `t3.medium` Spot + prefix delegation | `t3.small`; 2 nodes | Fits ~20 pods in 4 GiB; Spot can be reclaimed | Node capacity `110`; `kubectl top nodes` |
| HPA | 1–3 at 70% CPU, no `replicas` in Deployment | Fixed replicas | Elastic; needs CPU requests and metrics-server | HPA events, Grafana graph |

For the M16 "commit triggers a sync" demo with autoscaling on, don't
change `backend.replicaCount` (the HPA owns it) — change something visible
such as a frontend replica count or a backend environment value.

## Data sources

| Source | Origin | Type |
|---|---|---|
| Task schema + 3 sample tasks | Reference app `init.sql` → `seed.sql` → chart `files/seed.sql` | Static, bounded |
| Demo tasks | Created live | Live, one session |
| Failure inputs | Forbidden secret, bad tag, deleted Deployment, missing CPU request | Deliberate |

## Project architecture

```text
gitlab.com repo ◄── Argo CD (pull, deploy token) ── EKS dcta-eks (1 × t3.medium Spot, prefix delegation)
  deploy/argocd/*.yaml        ├─ external-secrets (IRSA dcta-eso-irsa) ─► Secrets Manager, SSM ─► db-credentials, db-config
  deploy/platform/            ├─ nginx-ingress ◄─ NLB (/32) ◄─ browser
  deploy/apps/task-app/       │     ├─ /api ─► backend (HPA 1–3) ─TLS─► RDS (private)
  deploy/charts/task-app/     │     ├─ /    ─► frontend
                              │     └─ grafana.dcta.test ─► Grafana
                              ├─ argocd
                              ├─ monitoring (kube-prometheus-stack)
                              └─ kube-system (metrics-server)
```

## Data model

`tasks` (UUID `id`, `title`, `description`, `status` in
`TODO|IN_PROGRESS|DONE`, `created_at`, `updated_at` via trigger).

## Technology stack

| Area | Technology | Module |
|---|---|---|
| IaC | Terraform 1.11+, `hashicorp/aws` 6.66.0, `hashicorp/helm` 3.3.0, `hashicorp/random` 3.9.1 | M1, M9, M11 |
| Foundation | VPC, ECR, RDS PostgreSQL 17, Secrets Manager, SSM | M1 |
| Cluster | EKS 1.36, `terraform-aws-modules/eks/aws` 21.26.0 | M9 |
| Identity | OIDC + IRSA | M10 |
| Secrets | External Secrets Operator 2.11.0 | M11 |
| Packaging | Helm `task-app` chart | M12 |
| Ingress | F5 NGINX Ingress Controller chart 2.7.3 (app 5.6.3) | M13 |
| GitOps | Argo CD chart 10.9.2 (app v3.5.3) | M14 |
| Observability/scaling | kube-prometheus-stack 91.8.1, metrics-server 3.14.0, HPA v2 | M15 |

## Key engineering features

Reference solution snippets. Stack skeletons and anything given in the
module labs (`versions.tf`, providers, remote state, ESO `helm_release`,
seed Job, metrics-server Application) are as in the labs.

**B-1 — `infra/40-track-b-cluster/eks.tf`**

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "21.26.0"

  name               = "dcta-eks"
  kubernetes_version = "1.36"

  vpc_id                   = local.f.vpc_id
  subnet_ids               = local.f.public_subnet_ids
  control_plane_subnet_ids = local.f.private_subnet_ids

  endpoint_public_access                   = true
  endpoint_public_access_cidrs             = [var.learner_cidr]
  enable_cluster_creator_admin_permissions = true

  # No control-plane logs: a daily-rebuilt cluster could otherwise leave an
  # untracked /aws/eks/dcta-eks/cluster log group behind on destroy.
  enabled_log_types           = []
  create_cloudwatch_log_group = false

  addons = {
    vpc-cni = {
      before_compute = true
      configuration_values = jsonencode({
        env = {
          ENABLE_PREFIX_DELEGATION = "true"
        }
      })
    }
    kube-proxy = {}
    coredns    = {}
  }

  eks_managed_node_groups = {
    spot = {
      ami_type       = "AL2023_x86_64_STANDARD"
      instance_types = ["t3.medium", "t3a.medium"]
      capacity_type  = "SPOT"

      min_size     = 1
      max_size     = 1
      desired_size = 1

      vpc_security_group_ids = [local.f.app_sg_id]
    }
  }
}

output "cluster_name" {
  value = module.eks.cluster_name
}

output "oidc_provider_arn" {
  value = module.eks.oidc_provider_arn
}

output "oidc_provider" {
  value = module.eks.oidc_provider
}
```

**B-2 — `infra/40-track-b-cluster/irsa.tf`**

```hcl
data "aws_caller_identity" "current" {}

locals {
  ssm_param_prefix = "arn:aws:ssm:${var.region}:${data.aws_caller_identity.current.account_id}:parameter"
}

data "aws_iam_policy_document" "eso_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [module.eks.oidc_provider_arn]
    }

    condition {
      test     = "StringEquals"
      variable = "${module.eks.oidc_provider}:sub"
      values   = ["system:serviceaccount:external-secrets:external-secrets"]
    }

    condition {
      test     = "StringEquals"
      variable = "${module.eks.oidc_provider}:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "eso" {
  name               = "dcta-eso-irsa"
  assume_role_policy = data.aws_iam_policy_document.eso_trust.json
}

data "aws_iam_policy_document" "eso_permissions" {
  statement {
    actions   = ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"]
    resources = [local.f.db_secret_arn]
  }

  statement {
    actions   = ["ssm:GetParameter"]
    resources = ["${local.ssm_param_prefix}/dcta/db/host", "${local.ssm_param_prefix}/dcta/db/name"]
  }
}

resource "aws_iam_role_policy" "eso" {
  name   = "read-db-config"
  role   = aws_iam_role.eso.id
  policy = data.aws_iam_policy_document.eso_permissions.json
}

output "eso_irsa_role_arn" {
  value = aws_iam_role.eso.arn
}
```

The SSM ARNs are built from the account and region rather than read from
`20-data`'s state, so the cluster stack doesn't depend on the data stack.
If ESO's logs ever show a denied action not listed here, add exactly that
action — don't widen to `*`.

**B-3 — ESO manifests**

`deploy/platform/cluster-secret-stores.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
---
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-parameter-store
spec:
  provider:
    aws:
      service: ParameterStore
      region: ap-southeast-1
```

`deploy/apps/task-app/external-secrets.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: task-app
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: task-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: DB_USER
      remoteRef:
        key: dcta/db/credentials
        property: username
    - secretKey: DB_PASSWORD
      remoteRef:
        key: dcta/db/credentials
        property: password
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-config
  namespace: task-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-parameter-store
  target:
    name: db-config
    creationPolicy: Owner
  data:
    - secretKey: DB_HOST
      remoteRef:
        key: /dcta/db/host
    - secretKey: DB_NAME
      remoteRef:
        key: /dcta/db/name
```

**B-4 — chart changes**

`deploy/charts/task-app/values-eks.yaml`:

```yaml
database:
  enabled: false

seed:
  enabled: true

ingress:
  enabled: true
  className: nginx

frontend:
  replicaCount: 1
  image:
    repository: <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/dcta-frontend
    tag: "<git-sha>"
  containerPort: 80
  service:
    type: ClusterIP
    port: 80
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      memory: 128Mi

backend:
  replicaCount: 1
  image:
    repository: <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/dcta-backend
    tag: "<git-sha>"
  containerPort: 3000
  service:
    type: ClusterIP
    port: 3000
  env:
    PORT: "3000"
    DB_PORT: "5432"
    DB_SSL_CA_PATH: /app/certs/rds-ca.pem
  envFromSecrets:
    - db-credentials
    - db-config
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      memory: 256Mi
  autoscaling:
    enabled: true
    minReplicas: 1
    maxReplicas: 3
    targetCPUUtilizationPercentage: 70
```

`templates/backend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: {{ .Release.Namespace }}
spec:
  {{- if not .Values.backend.autoscaling.enabled }}
  replicas: {{ .Values.backend.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
          ports:
            - containerPort: {{ .Values.backend.containerPort }}
          env:
          {{- range $name, $value := .Values.backend.env }}
            - name: {{ $name }}
              value: {{ $value | quote }}
          {{- end }}
          {{- with .Values.backend.envFromSecrets }}
          envFrom:
          {{- range . }}
            - secretRef:
                name: {{ . }}
          {{- end }}
          {{- end }}
          readinessProbe:
            httpGet:
              path: /health
              port: {{ .Values.backend.containerPort }}
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: {{ .Values.backend.containerPort }}
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            {{- toYaml .Values.backend.resources | nindent 12 }}
```

`templates/ingress.yaml`:

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: task-app
  namespace: {{ .Release.Namespace }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: {{ .Values.backend.service.port }}
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: {{ .Values.frontend.service.port }}
{{- end }}
```

`templates/hpa.yaml`:

```yaml
{{- if .Values.backend.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend
  namespace: {{ .Release.Namespace }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: {{ .Values.backend.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.backend.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.backend.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

The frontend Deployment/Service follow the same pattern (port 80, no
`envFrom`); the local `database` templates are wrapped in
`{{- if .Values.database.enabled }}`.

**B-5 — `infra/41-track-b-platform/ingress-and-argocd.tf`**

```hcl
resource "helm_release" "nginx_ingress" {
  name             = "nginx-ingress"
  repository       = "https://helm.nginx.com/stable"
  chart            = "nginx-ingress"
  version          = "2.7.3"
  namespace        = "nginx-ingress"
  create_namespace = true
  wait             = true
  timeout          = 900

  values = [yamlencode({
    controller = {
      allowEmptyIngressHost = true
      ingressClass = {
        name                = "nginx"
        create              = true
        setAsDefaultIngress = false
      }
      service = {
        type = "LoadBalancer"
        annotations = {
          "service.beta.kubernetes.io/aws-load-balancer-type" = "nlb"
        }
        loadBalancerSourceRanges = [var.learner_cidr]
      }
      resources = {
        requests = { cpu = "50m", memory = "128Mi" }
      }
    }
  })]
}

resource "helm_release" "argocd" {
  name             = "argocd"
  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argo-cd"
  version          = "10.9.2"
  namespace        = "argocd"
  create_namespace = true
  wait             = true
  timeout          = 900

  values = [yamlencode({
    dex           = { enabled = false }
    notifications = { enabled = false }
  })]
}
```

**B-6 — `deploy/argocd/` Applications** (repo URL placeholder)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.com/<group>/<project>.git
    targetRevision: main
    path: deploy/platform
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: task-app-secrets
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.com/<group>/<project>.git
    targetRevision: main
    path: deploy/apps/task-app
  destination:
    server: https://kubernetes.default.svc
    namespace: task-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: task-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.com/<group>/<project>.git
    targetRevision: main
    path: deploy/charts/task-app
    helm:
      valueFiles:
        - values-eks.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: task-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**B-7 — `deploy/argocd/kube-prometheus-stack.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 91.8.1
    helm:
      valuesObject:
        alertmanager:
          enabled: false
        kubeControllerManager:
          enabled: false
        kubeScheduler:
          enabled: false
        kubeEtcd:
          enabled: false
        kubeProxy:
          enabled: false
        prometheusOperator:
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
        prometheus:
          prometheusSpec:
            retention: 6h
            resources:
              requests:
                cpu: 100m
                memory: 400Mi
              limits:
                memory: 800Mi
        grafana:
          admin:
            existingSecret: grafana-admin
            userKey: admin-user
            passwordKey: admin-password
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
          ingress:
            enabled: true
            ingressClassName: nginx
            hosts:
              - grafana.dcta.test
            path: /
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

## Validation & testing

Expected results (illustrative — replace with captured output from a
reference run before the cohort starts).

| Check | Expected |
|---|---|
| `nc` to RDS from laptop | Times out |
| NAT count | `0` |
| Node | 1 × `t3.medium`/`t3a.medium`, `SPOT`, `capacity.pods` = `110` |
| `pg_isready` from pod | `<host>:5432 - accepting connections` |
| IRSA allowed pod | `arn:aws:sts::<ACCOUNT_ID>:assumed-role/dcta-eso-irsa/…`, `dcta/db/credentials`, then `AccessDeniedException` for `dcta/other-team/db` |
| IRSA `default` pod | `Unable to locate credentials` (IMDS blocked for pods) or `AccessDenied` as the node role |
| ESO | Stores `Valid`/`True`; ExternalSecrets `SecretSynced` |
| ESO forbidden | Status `SecretSyncedError` with an access-denied message; existing `Secret` keeps its data |
| Routing | `200`; `3` |
| Backend at 0 | `502`/`503` from NGINX on `/api/tasks`; `200` on `/` |
| GitOps commit | `OutOfSync` → `Synced` within ~3 minutes |
| Self-heal | Deployment recreated within seconds |
| Bad commit | Application `Degraded`, old ReplicaSet pod still `Running`; `git revert` → `Healthy` |
| HPA | Events `New size: 2`, `New size: 3`; later `New size: 1` (after ~5 min stabilization) |
| Missing CPU request | `TARGETS <unknown>/70%`; event `failed to get cpu utilization: missing request for cpu` |
| Teardown | `describe-load-balancers` → `0` |

## Output & usage notes

- NLB hostname and IPs change each session; `grafana.dcta.test` hosts-file
  entries must be updated and removed afterwards.
- Self-heal reverts manual edits — expect learners to be surprised once.
- HPA scale-down takes ~5 minutes after load stops (default stabilization
  window); allow for it in the 15-minute defense.

## Repository structure

```text
<learner-project>/
├── app/                               # reference app + M1 changes
├── infra/
│   ├── backend.hcl, env.sh
│   ├── 10-foundation/  20-data/
│   ├── 40-track-b-cluster/            # eks.tf, irsa.tf, remote state
│   └── 41-track-b-platform/           # external-secrets, nginx-ingress, argocd helm_releases
├── deploy/
│   ├── argocd/                        # platform, task-app-secrets, task-app, metrics-server, kube-prometheus-stack
│   ├── platform/                      # ClusterSecretStores
│   ├── apps/task-app/                 # Namespace + ExternalSecrets
│   └── charts/task-app/               # chart, values.yaml, values-eks.yaml, files/seed.sql
└── docs/capstone/                     # evidence + decision-matrix.md
```

## Documentation

Gate evidence is identical to the cohort list. Check it's real output and
that the own-words explanation covers: foundation persists; apply order
`10 → 20 → 40 → 41 → kubectl apply deploy/argocd`; destroy order
`41 → 40 → 20`; why the platform stack goes first (the NLB is created by a
Kubernetes Service and must be removed while the cluster still exists).

## Checkpoint (self-assessed)

With expected answers:

- [ ] 1–6. Foundation — as in the Track A instructor checkpoint items 1–6.
- [ ] 7. `aws eks describe-cluster` → version `1.36`; `publicAccessCidrs` = learner `/32`.
- [ ] 8. One node, `SPOT`, capacity `110`; `pg_isready` accepting.
- [ ] 9. Trust `sub` exact match; allowed/denied logs as above.
- [ ] 10. `helm list -n external-secrets` shows chart `external-secrets-2.11.0`.
- [ ] 11. Two stores, two `SecretSynced`; forbidden test shows access denied.
- [ ] 12. `helm template … -f values-eks.yaml` renders no PostgreSQL; seed Job `Complete`.
- [ ] 13. One NLB; controller args include `-allow-empty-ingress-host`.
- [ ] 14. `/api` → backend; backend-at-0 gives 5xx.
- [ ] 15. No `argocd-server` LoadBalancer Service; token not in Git.
- [ ] 16. Three Applications `Synced/Healthy`; self-heal and revert proven.
- [ ] 17. `kube-prometheus-stack` `Synced/Healthy`; Grafana login with the shell-created password.
- [ ] 18. HPA events 1 → 3 → 1; Grafana graph; `<unknown>` failure shown.
- [ ] 19. Decision matrix ≥ 5 rows, defended.
- [ ] 20. `describe-load-balancers` → `0` after destroy.

## Project scope

Does: one app on a one-node EKS cluster, single account/region, GitOps
for everything in-cluster, secrets from AWS, one load balancer, HPA,
daily rebuild/teardown.

Does not: Route 53/HTTPS, node autoscaling or multiple nodes, private-subnet
nodes with NAT, persistence/backups, multi-region, alerting, log
aggregation, app code changes beyond M1.

**Instructor notes — where the course differs from the original syllabus
and ARCH v6.0** (do not share with learners):

- **F5 NGINX Ingress Controller instead of ingress-nginx.** The community
  ingress-nginx project was retired in March 2026 (no further security
  fixes). Amends ADR-DCTA-AWS-E003-06; same single-NLB design.
- **Host-less Ingress limitation.** F5 NIC allows only one host-less
  Ingress, so Grafana uses host `grafana.dcta.test` instead of ARCH's
  `/grafana` path.
- **`t3.medium` Spot + VPC CNI prefix delegation.** ADR-DCTA-AWS-E003-02's
  `t3.medium` is still pending sponsor approval; even at `t3.medium` the
  default 17-pod limit is below the ~20 pods the M15 stack needs, so prefix
  delegation (110 pods) was added. If sponsors keep `t3.small`, trim M15
  (drop node-exporter/kube-state-metrics and lower requests) and re-test.
- **`eksctl` read-only;** teardown is `terraform destroy` only
  (ADR-DCTA-AWS-E003-03).
- **No wrapper scripts;** daily commands are explicit.
