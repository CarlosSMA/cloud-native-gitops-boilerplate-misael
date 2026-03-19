# Cloud Native Experience: System Architecture

> **Scope**: This document describes the system architecture of the Operating Kit as shipped today on AWS. Azure and GCP support is **(Roadmap)**. The HTML architecture diagram (`cloud-native-architecture.html`) is the visual reference for this document.

## Architecture Overview

The Operating Kit is a layered system that provisions, deploys, and operates Liferay DXP on Kubernetes. Four technologies form the foundation: Terraform bootstraps the EKS cluster and installs platform services, Crossplane manages per-environment cloud resources as Kubernetes CRDs, ArgoCD continuously reconciles infrastructure and application state from Git, and Helm charts package the application with cloud-specific extensions. Traffic enters through AWS Shield and an Envoy Gateway before reaching isolated per-namespace Liferay workloads. AWS managed services (RDS, OpenSearch, S3) are provisioned per environment with full isolation.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ACTORS & SOURCES                                                               │
│                                                                                 │
│  DevOps Team   Infra Team   Deployments  Obs. Console │ End Users │ Customer   │
│  (Git PR,      (kubectl,    (ArgoCD UI)  (Grafana     │ (Browser, │ Repo       │
│   ArgoCD UI)    tf apply)                 Dashboards)  │  HTTPS)   │ (Helm,     │
│                                                        │           │  ArgoCD)   │
│                                                        │           │ Customer CI│
│                                                        │           │ Customer   │
│                                                        │           │ Registry   │
│                                                        │           │ (e.g. ECR) │
└───────────────────────────────────┬─────────────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────────────────────┐
│  DDoS PROTECTION + CDN  ·  AWS Shield + CloudFront  ·  Customer-managed          │
└───────────────────────────────────┬─────────────────────────────────────────────┘
                                    │ HTTPS
┌───────────────────────────────────▼─────────────────────────────────────────────┐
│  HTTP(S) LOAD BALANCER + GATEWAY API                                            │
│                                                                                 │
│  AWS NLB ──→ Envoy Gateway (in-cluster)                  cert-manager (Roadmap) │
│              TLS termination · per-namespace routing                             │
│              HTTP→HTTPS · Proxy Protocol v2                                     │
│              BYO certs via ESO (1h refresh)                                     │
└──────────┬──────────────────────┬───────────────────────┬───────────────────────┘
           │                      │                       │ per-namespace routing
┌──────────▼──────────────────────▼───────────────────────▼───────────────────────┐
│  EKS KUBERNETES CLUSTER                                                         │
│                                                                                 │
│  ┌─── PLATFORM SERVICES (cluster-wide) ──────────────────────────────────────┐  │
│  │ ArgoCD        ESO           Crossplane      Argo Workflows                │  │
│  │ GitOps        Secrets Sync  Cloud Resources  Backup/Restore               │  │
│  │ Engine        (IRSA)                                                      │  │
│  │                                                                           │  │
│  │ Grafana Alloy              KEDA              Kubernetes Namespace         │  │
│  │ Metrics DaemonSet          Event-driven HPA  liferay-infra                │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌─ liferay-dev ──────┐  ┌─ liferay-uat ──────┐  ┌─ liferay-prd ──────────┐  │
│  │ Liferay DXP        │  │ Liferay DXP        │  │ Liferay DXP            │  │
│  │ StatefulSet 1–3    │  │ StatefulSet 1–3    │  │ StatefulSet 1–3        │  │
│  │ HPA · IRSA         │  │ HPA · IRSA         │  │ HPA · IRSA             │  │
│  │ non-root           │  │ non-root           │  │ non-root               │  │
│  │                    │  │                    │  │                        │  │
│  │ ↓ RDS·OpenSearch·S3│  │ ↓ RDS·OpenSearch·S3│  │ ↓ RDS·OpenSearch·S3   │  │
│  └────────────────────┘  └────────────────────┘  └────────────────────────┘  │
└───────────────────────────────────┬─────────────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────────────────────┐
│  AWS MANAGED CLOUD SERVICES                                                     │
│                                                                                 │
│  PER-ENVIRONMENT (isolated)        SHARED SERVICES       OBSERVABILITY          │
│  ┌───────────┐ ┌───────────┐      ┌───────────┐        ┌───────────┐           │
│  │ RDS       │ │ OpenSearch│      │ Secrets   │        │ Prometheus│           │
│  │ PostgreSQL│ │ Full-text │      │ Manager   │        │ AMP       │           │
│  │ PITR      │ │ search    │      │ KMS · ESO │        │ 15-day    │           │
│  └───────────┘ └───────────┘      │ source 1h │        └───────────┘           │
│  ┌───────────┐ ┌───────────┐      └───────────┘        ┌───────────┐           │
│  │ S3 Buckets│ │ AWS Backup│      ┌───────────┐        │ Grafana   │           │
│  │ 3 per env │ │ Snapshots │      │ CloudWatch│        │ AMG       │           │
│  │ docs·wksp │ │ Argo      │      │ Logs·Met. │        │ JVM · DB  │           │
│  │ · backups │ │ restore   │      │ cluster   │        │ dashboards│           │
│  └───────────┘ └───────────┘      └───────────┘        └───────────┘           │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Actors and Sources

### Operator Consoles

| Actor | Interface | Role |
|-------|-----------|------|
| **DevOps Team** | Git PR, ArgoCD UI | Manages application configuration, promotes releases across environments |
| **Infra Team** | kubectl, `tf apply` | Manages cloud infrastructure, Kubernetes clusters, platform services |
| **Deployments** | ArgoCD UI | Monitors and manages ArgoCD application deployments |
| **Obs. Console** | Grafana Dashboards | Monitors JVM health, database connections, and Kubernetes resources |

### End Users

| Actor | Interface | Role |
|-------|-----------|------|
| **End Users** | Browser (HTTPS) | Accesses Liferay DXP through the load balancer |

### Customer-Owned Sources

| Source | Interface | Role |
|--------|-----------|------|
| **Customer Repo** | Helm charts, ArgoCD Applications | Single source of truth for cluster and application desired state |
| **Customer CI** | Build, push | Builds container images, pushes to the customer's container registry, creates PRs to the customer repo |
| **Customer Registry** | Container images (e.g. ECR) | Stores built images. Customers bring their own registry; AWS deployments typically use ECR with IRSA authentication |

## Edge and Ingress Layer

### DDoS Protection and CDN

AWS Shield provides DDoS protection at the edge. Amazon CloudFront provides CDN and edge caching. Both layers are customer-managed and sit outside the Kubernetes cluster.

### Load Balancer and Gateway

Traffic enters the cluster through an **AWS Network Load Balancer** fronting an **Envoy Gateway** running in-cluster. Envoy Gateway implements the Kubernetes Gateway API and handles:

- **TLS termination**: Terminates HTTPS at the gateway, offloading encryption from application pods
- **Per-namespace routing**: Routes requests to the correct environment (dev, uat, prd) based on hostname or path rules
- **HTTP to HTTPS redirect**: Forces all plaintext connections to TLS
- **Proxy Protocol v2**: Preserves client IP addresses through the NLB
- **BYO certificates**: Customers supply their own TLS certificates via ESO, refreshed hourly from the cloud vault

**cert-manager** integration for automated Let's Encrypt certificates is **(Roadmap)**.

## EKS Kubernetes Cluster

The cluster hosts two layers: cluster-wide platform services and per-environment application namespaces.

### Platform Services

Six platform services run cluster-wide, installed by Terraform during bootstrap.

| Service | Function | Details |
|---------|----------|---------|
| **ArgoCD** | GitOps engine | Watches the GitOps repo, continuously reconciles desired state to the cluster. Deploys both infrastructure (Crossplane resources) and application (Liferay Helm charts) via ApplicationSets. |
| **ESO** | Secrets sync | External Secrets Operator bridges the cloud vault (AWS Secrets Manager) with Kubernetes Secrets. Authenticates via IRSA. Refreshes secrets on a 1-hour cycle. |
| **Crossplane** | Cloud resource provisioning | Manages per-environment AWS resources (RDS, OpenSearch, S3) as Kubernetes CRDs. Deployed via ArgoCD after Terraform bootstraps the controller and IRSA roles. |
| **Argo Workflows** | Backup, restore | Executes imperative DAG workflows for disaster recovery restores and cleanup. Not GitOps-driven: workflows run once and produce a side effect. |
| **Grafana Alloy** | Metrics collection (DaemonSet) | Runs on every node. Scrapes JMX metrics from Liferay pods on port 9404, collects Kubernetes metrics, and forwards to Prometheus (AMP). |
| **KEDA** | Event-driven HPA | Scales Liferay pods based on Prometheus queries. Replaces static HPA thresholds with metric-driven scaling decisions. |

### Environment Namespaces

Each environment runs in an isolated Kubernetes namespace. ArgoCD ApplicationSets dynamically create and manage these namespaces.

| Namespace | Replica Count | Notes |
|-----------|---------------|-------|
| `liferay-dev` | StatefulSet, 1–3 pods | Development environment |
| `liferay-uat` | StatefulSet, 1–3 pods | User acceptance testing |
| `liferay-prd` | StatefulSet, 1–3 pods | Production |

Every namespace runs Liferay DXP as a **StatefulSet** with:

- **HPA**: Horizontal Pod Autoscaler (KEDA-driven in production)
- **IRSA**: IAM Roles for Service Accounts, scoping each pod's AWS permissions to its own environment's resources
- **Non-root containers**: Pods run as non-root for security hardening

## AWS Managed Cloud Services

### Per-Environment Services (Isolated Instances)

Each environment receives dedicated, isolated instances of every data service. This prevents cross-environment data leakage and enables environment-specific retention policies.

| Service | Type | Per-Environment Details |
|---------|------|------------------------|
| **RDS** | PostgreSQL | Dedicated instance per environment. Supports Point-in-Time Recovery (PITR). |
| **OpenSearch** | AWS Managed | Dedicated domain per environment. Provides full-text search for Liferay content. |
| **S3 Buckets** | Object storage, 3 per environment | `documents` (portal content), `workspace` (CI/CD artifacts), `backups` (snapshots). IAM scoped per environment. |
| **AWS Backup** | Snapshots | Daily snapshots of RDS, S3, and OpenSearch. Argo Workflows uses these as restore targets. |

### Shared Services

| Service | Function | Details |
|---------|----------|---------|
| **AWS Secrets Manager** | Secret vault | Source of truth for all secrets. ESO reads from it on a 1-hour cycle. Encrypted with AWS KMS. |
| **CloudWatch** | Logs and metrics | Cluster-wide log aggregation and AWS-level metrics. |

### Observability Stack

| Service | Function | Details |
|---------|----------|---------|
| **Prometheus (AMP)** | Metrics storage | Amazon Managed Prometheus. 15-day retention. Receives metrics from Grafana Alloy. Accessible on :9090. |
| **Grafana (AMG)** | Dashboards | Amazon Managed Grafana. Pre-built dashboards for JVM health, database connections, and Kubernetes resources. Queries Prometheus as the data source. |

## Deployment Workflows

Three deployment workflows support different speed and flexibility tradeoffs.

### Workflow A: Custom Image (Fast Startup)

1. DevOps team pushes code to the customer repo
2. Customer CI builds a container image and pushes it to the customer registry
3. Customer CI creates a PR to the customer repo with the new image tag
4. Infra team reviews and merges the PR
5. ArgoCD detects the change and syncs the Helm release
6. StatefulSet performs a rolling update (fast startup, no bundle download)

### Workflow B: ConfigMap + S3 Bundle (Faster CI, Slower Startup)

1. DevOps team pushes code to the customer repo
2. Customer CI builds a bundle tarball and uploads it to S3
3. Customer CI creates a PR to the customer repo with the new bundle location
4. Infra team reviews and merges the PR
5. ArgoCD syncs; an initContainer downloads the bundle before Liferay starts
6. Liferay pod starts with the bundle mounted to `/deploy`

### Workflow C: S3 Extensions Hot-Deploy (No Restart)

1. DevOps uploads a JAR or LPKG directly to the S3 extensions bucket
2. The S3 CSI driver mounts the bucket as a ReadWriteMany PVC
3. Liferay detects the new file and hot-deploys it automatically
4. No image rebuild, no pod restart, no PR required

Best for rapid iteration, marketplace LPKGs, and OSGi configuration bundles.

## Secrets Management

ESO bridges the cloud vault with Kubernetes. The flow:

1. Secrets are stored in **AWS Secrets Manager** (or Azure Key Vault / GCP Secret Manager for other clouds)
2. A **ClusterSecretStore** configures ESO to authenticate via IRSA
3. **ExternalSecret** resources in each namespace declare which vault paths to sync
4. ESO fetches secrets on a 1-hour cycle and creates standard Kubernetes Secrets
5. Liferay pods mount these secrets as environment variables or volume mounts

Secrets are scoped by environment. The vault path convention is `liferay/{env}/{secret-name}`. An ExternalSecret in `liferay-dev` cannot access secrets under `liferay-prd`.

Customers can bring their own vault (HashiCorp Vault, 1Password, or cross-cloud configurations) by updating the Terraform provider configuration.

## Backup and Restore

Backup and restore runs as an **Argo Workflows** DAG, not through GitOps. A restore is a recovery action: it runs once, produces a side effect (restored data), and completes. ArgoCD resumes normal reconciliation after the restore finishes.

### Restore DAG Structure

```
get-current-infrastructure-state ──┐
                                    ├──→ restore-rds-instance ──┐
get-peer-recovery-points ──────────┘                            │
                                    ├──→ restore-s3-bucket ─────┤
                                    │                            │
                                    │                            ▼
                                    │                      restart-liferay
                                    │                            │
                                    │                    ┌───────┴───────┐
                                    │                    ▼               ▼
                                    │          clean-up-rds      clean-up-s3
```

- The first two steps run in parallel to gather infrastructure state and identify matching recovery points
- RDS and S3 restores run in parallel
- Cleanup removes superseded resources
- If the restore fails, the original resources remain untouched (automatic rollback)

The DevOps team triggers restores via `kubectl` (Argo CLI) or the Argo Workflows UI. Access is namespace-scoped: authorization to restore in dev does not grant access to restore in production.

## Key Architectural Principles

1. **GitOps-driven**: All deployments (infrastructure and application) are managed through Git commits and ArgoCD reconciliation. Imperative operations (restore, migration) use Argo Workflows.
2. **Environment isolation**: Separate namespaces (`liferay-{env}`), separate cloud resources (RDS, OpenSearch, S3), and separate IAM scopes per environment.
3. **Cloud-agnostic design**: Helm chart layering (`default/` extended by `aws/`) enables portability. Azure and GCP wrappers are **(Roadmap)**.
4. **Two-layer infrastructure as code**: Terraform bootstraps the cluster and installs platform services. Crossplane manages per-environment cloud resources at runtime.
5. **Secrets as single source of truth**: Connection strings, credentials, and licenses live in the cloud vault. ESO syncs them to Kubernetes. No secrets in Git.
6. **Observability first**: Grafana Alloy collects JVM and Kubernetes metrics from every node. Prometheus (AMP) stores them. Grafana (AMG) visualizes them. KEDA uses them for scaling.
7. **Separation of concerns**: The Infra team owns the platform (Terraform, Helm charts, Crossplane compositions). The DevOps team owns customizations and promotes releases via PRs to the customer repo. ArgoCD enforces the boundary.

## Cloud Provider Support

| Provider | Status | Notes |
|----------|--------|-------|
| **AWS** | Shipped | EKS, RDS, OpenSearch, S3, Secrets Manager, AMP, AMG, AWS Backup, CloudWatch |
| **Azure** | **(Roadmap)** | AKS, Azure Database, Azure Key Vault, ACR |
| **GCP** | **(Roadmap)** | GKE, Cloud SQL, GCP Secret Manager, GCR |

## Connection Legend

| Flow | Description |
|------|-------------|
| HTTPS traffic | End users to load balancer |
| Ingress routing | Envoy Gateway to per-namespace Liferay pods |
| GitOps / ArgoCD | ArgoCD watches customer repo, deploys to namespaces |
| CI · image push | Customer CI pushes images to customer registry |
| Crossplane provisions | Crossplane creates/manages RDS, OpenSearch, S3 |
| ESO syncs | ESO reads from Secrets Manager, creates Kubernetes Secrets |
| Argo Workflows | Backup/restore DAGs targeting AWS Backup snapshots |
