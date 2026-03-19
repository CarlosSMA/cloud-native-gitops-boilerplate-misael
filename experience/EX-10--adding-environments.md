# EX-10: Adding Environments

**Arc:** Operate
**Persona:** DevOps / Platform Engineer
**Time:** ~5 minutes (Git change) + 45 minutes (managed services provisioning)
**Validation Status:** Not validated

## Product Outcome

Adding a new environment (UAT, staging, perf-test) is creating a directory with two YAML files. ArgoCD auto-discovers it via the ApplicationSet git generator. Crossplane provisions dedicated managed services. No Terraform re-run. Non-production environments cost $0 in licensing.

## User Journey

### Step 1: Create Environment Directory

**Action:** Create directory and files, commit, push.

```
liferay/projects/default/environments/uat/
  infrastructure.yaml
  liferay.yaml
```

`infrastructure.yaml`:
```yaml
database:
  instanceClass: db.t3.medium
  storageGB: 20
search:
  opensearch:
    instanceType: r6g.large.search
    instanceCount: 2
```

`liferay.yaml`:
```yaml
liferay-default:
  replicaCount: 1
  resources:
    requests:
      cpu: "4"
      memory: 4Gi
    limits:
      cpu: "4"
      memory: 4Gi
```

### Step 2: ArgoCD Auto-Discovery (Automatic)

**System response:** ApplicationSet detects new path `liferay/projects/default/environments/uat/`. Creates:
- `default-uat-infra` (ArgoCD Application)
- `default-uat-app` (ArgoCD Application)
- Namespace `liferay-default-uat` (auto-created)

**Test criteria:**
- [ ] Applications appear in ArgoCD within 3 minutes of push
- [ ] Namespace created automatically
- [ ] No Terraform changes required
- [ ] No manual ArgoCD configuration

### Step 3: Managed Services Provision (Automatic)

Same as EX-03. RDS, OpenSearch, and S3 provision for the new environment. Takes 20-45 minutes.

**Test criteria:**
- [ ] Dedicated RDS instance for UAT (not shared with dev)
- [ ] Dedicated OpenSearch domain for UAT
- [ ] Dedicated S3 buckets for UAT
- [ ] `managed-service-details` secret created in `liferay-default-uat` namespace
- [ ] Liferay pod starts and becomes Ready

### Step 4: Remove Environment

**Action:** Delete the environment directory, commit, push.

**System response:** ArgoCD detects removal. With pruning enabled, deletes ArgoCD applications. Crossplane deletes managed resources (if `deletionProtection: false`).

**Test criteria:**
- [ ] ArgoCD apps removed
- [ ] Crossplane resources deleted (check AWS Console)
- [ ] Namespace cleaned up
- [ ] No orphaned AWS resources

## Financial Impact

Each non-production environment costs $0 in Liferay licensing. AWS infrastructure costs only (RDS, OpenSearch, S3). The ability to spin up environments on demand eliminates the pressure to share environments across teams.

## Future Improvements

- Environment templates (pre-configured sizing tiers: small, medium, large)
- Cost estimate before provisioning
- Ephemeral environments for PR previews
