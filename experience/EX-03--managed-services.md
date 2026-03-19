# EX-03: Managed Services Provisioning

**Arc:** Get Running
**Persona:** Platform Engineer (monitoring) / Automatic
**Time:** 20-45 minutes (runs automatically after EX-02)
**Validation Status:** Validated (with issues)

## Product Outcome

Crossplane automatically provisions all managed services (RDS PostgreSQL, OpenSearch, S3 buckets) per environment. Credentials are auto-generated and injected as Kubernetes Secrets. The platform engineer waits and monitors, no manual action required.

## User Journey

### Step 1: ArgoCD Discovers Environments (Automatic)

**Trigger:** Phase 3 completion. ArgoCD scans GitOps repo at `liferay/projects/*/environments/*`.

**System creates ArgoCD Applications:**
- `default-dev-infra` and `default-dev-app`
- `default-prd-infra` and `default-prd-app`
- (one pair per environment directory detected)

**What the user sees in ArgoCD UI:** Four applications appear. Infrastructure apps start syncing. Application apps show Progressing (waiting on managed services).

**Test criteria:**
- [ ] ArgoCD creates the expected number of applications (2 per environment)
- [ ] Namespace `liferay-default-dev` (and prd) is created automatically
- [ ] Infrastructure app begins syncing immediately

### Step 2: Crossplane Creates AWS Resources (Automatic)

**Trigger:** ArgoCD syncs the `liferay-aws-infrastructure` chart, creating a `LiferayInfrastructure` CR.

**System creates per environment:**

| Resource | Time | Purpose |
|----------|------|---------|
| SecurityGroup (x2) | < 1 min | VPC-scoped DB and search access |
| SecurityGroupIngressRule (x2) | < 1 min | Allow traffic within VPC CIDR |
| SubnetGroup | < 1 min | Multi-AZ placement for RDS |
| S3 Bucket (doclib) | < 1 min | Document library storage |
| S3 Bucket (overlay) | < 1 min | CI/CD overlay storage |
| IAM User + AccessKey | < 1 min | CI uploader credentials |
| IAM Policies (x3) | < 1 min | Scoped access per resource |
| RDS Instance | 15-20 min | PostgreSQL database |
| OpenSearch Domain | 20-30 min | Search engine |
| Kubernetes Secret | After all above | `managed-service-details` |

**What the user monitors:**
```bash
kubectl get liferayinfrastructures --all-namespaces
# READY=True means all services provisioned

kubectl get managed -A | grep -v True
# Shows resources still provisioning
```

**Test criteria:**
- [ ] All Crossplane managed resources reach SYNCED=True
- [ ] All Crossplane managed resources reach READY=True
- [ ] `managed-service-details` secret is created in each environment namespace
- [ ] Secret contains keys: DATABASE_ENDPOINT, DATABASE_USERNAME, DATABASE_PASSWORD, SEARCH_ENDPOINT, STORAGE_BUCKET_NAME (and others)
- [ ] S3 bucket names are globally unique (ISS-A: collision risk with generic names)
- [ ] SecurityGroupIngressRules resolve their parent SecurityGroup (L-015: fails with deletionProtection)

### Step 3: Credentials Auto-Generated (Automatic)

**System behavior:** Crossplane generates 16-character passwords with special characters for database and search. These are written to the `managed-service-details` Kubernetes Secret. They never appear in Git, environment variables, or Terraform state.

**Test criteria:**
- [ ] Credentials are not in any Git commit
- [ ] Credentials are not in Terraform state
- [ ] `managed-service-details` secret has all required keys
- [ ] Liferay pods can connect to RDS using these credentials
- [ ] Liferay pods can connect to OpenSearch using these credentials

## Waiting Period Expectations

The user should expect:
- Liferay pods in `CrashLoopBackOff` or `Pending` during this phase (normal)
- Dev and prd may provision sequentially, not in parallel (L-012)
- Total time for 2 environments: 45-60 minutes (not 15-20 as sometimes implied)

## Known Issues

| ID | Issue | Severity | Workaround |
|----|-------|----------|------------|
| L-004 | OpenSearch instanceCount=1 invalid with 2-AZ | Blocks | Set to 2 or 4 |
| L-012 | Crossplane provisions sequentially, not in parallel | Degrades | Wait (45-60 min for 2 envs) |
| L-015 | deletionProtection=true breaks reference resolution | Critical | Set to false |
| ISS-A | S3 naming collision on generic project names | Blocks | Use unique project/env names |
| ISS-C | Liferay may start on Hypersonic if secret arrives late | Degrades | Restart pod after secret exists |

## Future Improvements

- Progress indicator showing which resources are still provisioning
- Parallel environment provisioning
- S3 naming that incorporates AWS account ID to prevent collisions
- deletionProtection fix (add LateInitialize to managementPolicies)
- Pod dependency on `managed-service-details` secret (don't start until it exists)
