# EX-02: Platform Provisioning

**Arc:** Get Running
**Persona:** Platform Engineer
**Time:** 20-40 minutes
**Validation Status:** Validated (with issues)

## Product Outcome

A single bootstrap command provisions the complete Kubernetes platform: EKS cluster, VPC networking, Envoy Gateway, ArgoCD, Crossplane, External Secrets Operator, and Argo Workflows. The platform engineer runs one command and gets a fully operational platform.

## User Journey

### Step 1: Run Bootstrap

**Action:** Execute the bootstrap script.

```bash
bash <(curl -sL https://raw.githubusercontent.com/liferay/liferay-portal/refs/heads/master/cloud/scripts/bootstrap.sh)
```

Or with a local config:

```bash
./cloud/scripts/setup_aws.sh ./config.json
```

**System response:** Script validates tools, downloads bootstrap archive from `cdn.liferay.cloud`, extracts to `/tmp/liferay-bootstrap.XXXXXX/`, generates `global_terraform.tfvars`.

**Test criteria:**
- [ ] Script validates all required tools before proceeding
- [ ] Error messages are clear when tools are missing
- [ ] `config.json` validation catches malformed input

### Step 2: Phase 1 - EKS (Terraform Apply)

**Action:** Script runs `terraform apply`. User confirms with `yes` (L-005: no `-auto-approve`).

**System creates:**
- VPC with public/private subnets (CIDR `10.0.0.0/16`)
- EKS cluster with Auto Mode (or managed node groups)
- StorageClass `gp3` (default)
- OIDC provider for IRSA
- IAM roles for all components (Crossplane providers, Liferay SA, Envoy)
- Envoy Gateway (2 replicas in `envoy-gateway-system`)
- CloudWatch Observability addon (Container Insights, 90-day retention)
- kubeconfig updated locally

**What the user sees:** Terraform output with resource counts. `kubectl get nodes` returns healthy nodes.

**Test criteria:**
- [ ] EKS cluster is reachable via kubectl
- [ ] All nodes are Ready
- [ ] StorageClass `gp3` is default
- [ ] Envoy Gateway pods are Running
- [ ] CloudWatch agent pods are Running

### Step 3: Phase 2 - GitOps Platform (Terraform Apply)

**Action:** Script continues. User confirms with `yes`.

**System creates:**

| Component | Namespace | Purpose |
|-----------|-----------|---------|
| ArgoCD | `argocd-system` | Continuous delivery |
| Crossplane | `crossplane-system` | AWS resource provisioning |
| External Secrets Operator | `external-secrets-system` | Vault-to-K8s secret sync |
| Argo Workflows | `argo-workflows-system` | Backup/restore orchestration |

**Test criteria:**
- [ ] All pods in `argocd-system` are Running
- [ ] All pods in `crossplane-system` are Running
- [ ] All pods in `external-secrets-system` are Running
- [ ] Argo Workflows pods are Running

### Step 4: Phase 3 - GitOps Resources (Terraform Apply)

**Action:** Script continues. User confirms with `yes`.

**System creates:**
- ArgoCD ApplicationSets (infrastructure + application, git generator)
- Crossplane Providers (RDS, S3, OpenSearch, EC2, IAM, Kubernetes)
- Crossplane XRD (`LiferayInfrastructure`) and Composition
- GatewayClass + EnvoyProxy (NLB: internet-facing, 3 replicas, HPA 3-10)
- ClusterSecretStore (ESO pointing to AWS Secrets Manager)
- ExternalSecret for GitOps credentials
- ArgoCD repo registration

**What the user sees:** Bootstrap script prints:
- ArgoCD admin password
- Port-forward command for ArgoCD UI

**Test criteria:**
- [ ] All Crossplane providers are Healthy
- [ ] ArgoCD can reach the GitOps repo (check ArgoCD repo settings)
- [ ] ApplicationSets are created (one per environment detected)
- [ ] ArgoCD admin password is printed clearly
- [ ] Port-forward command works (L-003: ArgoCD health checks may show Degraded)

### ArgoCD Console Access (Bootstrap Output)

The bootstrap script outputs everything needed to access ArgoCD. This is the first UI the platform engineer interacts with.

**Retrieve credentials:**
```bash
kubectl get secret argocd-initial-admin-secret -n argocd-system \
  -o jsonpath="{.data.password}" | base64 -d
```

**Access via port-forward:**
```bash
kubectl port-forward service/argocd-server 9443:443 -n argocd-system
# WSL: add --address 0.0.0.0 and use WSL IP from Windows browser
```

**Access:** `https://localhost:9443`, login `admin` / (password from above).

**What the user sees in ArgoCD:**

| Application | Shows |
|-------------|-------|
| `default-dev-infra` | Crossplane-managed RDS, OpenSearch, S3 resources |
| `default-dev-app` | Liferay StatefulSet, Service, HTTPRoute |
| `default-prd-infra` | Same as dev, production sizing |
| `default-prd-app` | Same as dev, production replicas |
| `liferay-infrastructure-provider` | Crossplane providers, compositions, XRDs |

**Optional: domain access** (after DNS/TLS in EX-05):
```hcl
# gitops/resources/terraform.tfvars
argocd_domain_config = {
  hostname                 = "argocd.example.com"
  tls_external_secret_name = "liferay/certificates/example-tls"
}
```

**Test criteria:**
- [ ] ArgoCD UI loads after port-forward
- [ ] All expected applications are visible
- [ ] Sync status is accurate
- [ ] Resource tree shows Crossplane managed resources under infra apps
- [ ] Diff view links changes to Git commits

## Known Issues

| ID | Issue | Severity | Workaround |
|----|-------|----------|------------|
| L-002 | Managed node groups need AWS LBC manually | Blocks | Install AWS LBC Helm chart |
| L-003 | ArgoCD has no health checks for Crossplane CRDs | Degrades | Add custom health checks to argocd-cm |
| L-005 | Bootstrap script lacks `-auto-approve` | Degrades | Manual `yes` at each phase |
| L-006 | `KUBE_CONFIG_PATH` doesn't persist between shells | Degrades | Pass inline with terraform |
| L-007/L-009 | StorageClass provisioner mismatch (Auto Mode vs managed) | Blocks | Recreate StorageClass |

## Future Improvements

- Single-command bootstrap with no interactive prompts
- Pre-configured ArgoCD health checks for Crossplane CRDs
- Validation that StorageClass provisioner matches CSI driver
- Progress indicator during Terraform phases
