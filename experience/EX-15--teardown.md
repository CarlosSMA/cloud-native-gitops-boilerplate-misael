# EX-15: Teardown

**Arc:** Operate
**Persona:** Platform Engineer
**Time:** 15-30 minutes
**Validation Status:** Not validated (known issues from prior deployments)

## Product Outcome

Full teardown follows the reverse of bootstrap: disable managed infrastructure, then destroy GitOps resources, GitOps platform, and EKS in order. Crossplane deletes AWS resources before providers are removed.

## User Journey

### Step 1: Disable Managed Infrastructure

**Action:** Edit each environment's `infrastructure.yaml`, commit, push.

```yaml
enabled: false
```

**Then in ArgoCD UI:** Sync each `*-infra` application with **Prune** enabled.

**System response:** Crossplane deletes RDS instances, OpenSearch domains, S3 buckets, security groups, IAM resources. This takes 10-15 minutes.

**Wait:** Verify in AWS Console that all managed resources are deleted before proceeding.

**Test criteria:**
- [ ] All Crossplane managed resources deleted
- [ ] No orphaned RDS instances in AWS
- [ ] No orphaned OpenSearch domains in AWS
- [ ] No orphaned S3 buckets in AWS
- [ ] deletionProtection=true resources require manual intervention or flag change first

### Step 2: Destroy GitOps Resources

**Action:**
```bash
cd /tmp/liferay-bootstrap.*/cloud/terraform/aws/gitops/resources
terraform destroy --auto-approve
```

**System response:** Removes ArgoCD ApplicationSets, Crossplane providers, GatewayClass, ClusterSecretStore, ExternalSecrets.

**Test criteria:**
- [ ] All Crossplane providers removed
- [ ] ArgoCD applications removed
- [ ] No stuck finalizers (ISS-F: known issue)

### Step 3: Destroy GitOps Platform

**Action:**
```bash
cd /tmp/liferay-bootstrap.*/cloud/terraform/aws/gitops/platform
terraform destroy --auto-approve
```

**System response:** Removes ArgoCD, Crossplane, ESO, Argo Workflows Helm releases and namespaces.

### Step 4: Destroy EKS Cluster

**Action:**
```bash
cd /tmp/liferay-bootstrap.*/cloud/terraform/aws/eks
terraform destroy --auto-approve
```

**System response:** Removes EKS cluster, VPC, subnets, NAT gateways, IAM roles, Envoy Gateway.

**Test criteria:**
- [ ] EKS cluster deleted from AWS Console
- [ ] VPC and all networking resources deleted
- [ ] IAM roles cleaned up
- [ ] No lingering CloudWatch log groups
- [ ] No lingering EBS volumes

## Known Issues

| ID | Issue | Severity | Workaround |
|----|-------|----------|------------|
| ISS-F | Stuck finalizers block namespace/resource deletion | Degrades | Manually patch finalizers to null |
| - | deletionProtection resources resist deletion | By design | Set deletionProtection: false first |
| - | Bootstrap temp directory may be lost | Degrades | Re-download bootstrap archive for Terraform state |

## Order Matters

Destroying out of order creates orphaned AWS resources that incur costs:

```
CORRECT:  Infrastructure disable -> Resources -> Platform -> EKS
WRONG:    EKS first -> Crossplane providers gone -> AWS resources orphaned
```

## Future Improvements

- Single teardown command (reverse of bootstrap)
- Pre-flight check for orphaned resources before EKS destruction
- Automated finalizer cleanup
- Terraform state preservation guidance
