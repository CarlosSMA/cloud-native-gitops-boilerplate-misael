# EX-01: Prerequisites and Setup

**Arc:** Get Running
**Persona:** Platform Engineer
**Time:** ~30 minutes
**Validation Status:** Validated (with issues)

## Product Outcome

The platform engineer has everything needed to run the bootstrap: a GitOps repo, cloud credentials, and a configuration file. The Operating Kit promises that this is the only manual setup required before a single command provisions the entire stack.

## User Journey

### Step 1: Fork the GitOps Boilerplate

**Action:** Fork `LiferayCloud/cloud-native-gitops-boilerplate` on GitHub.

**System response:** GitHub creates a copy of the repo with the standard directory structure.

**What the user sees:**
```
liferay/
  projects/default/
    base/
      infrastructure.yaml
      liferay.yaml
    environments/
      dev/
        infrastructure.yaml
        liferay.yaml
      prd/
        infrastructure.yaml
        liferay.yaml
  system/
    infrastructure-provider.yaml
charts/
  liferay-aws-infrastructure/
  liferay-aws-infrastructure-provider/
```

**Test criteria:**
- [ ] Repo contains `liferay/system/infrastructure-provider.yaml` (L-001: currently missing from boilerplate)
- [ ] Base `liferay.yaml` has a valid image tag
- [ ] Environment `infrastructure.yaml` files have valid defaults
- [ ] OpenSearch `instanceCount` is even (L-004: default of 1 is invalid with 2-AZ)

### Step 2: Install Local Tools

**Action:** Install `aws` CLI, `terraform`, `kubectl`, `git`, `curl`, `jq`, `tar`.

**What the user sees:** Tool version checks pass.

**Test criteria:**
- [ ] Documentation lists exact version requirements
- [ ] Documentation lists install commands per OS

### Step 3: Configure AWS Credentials

**Action:** Run `aws configure sso` and `export AWS_PROFILE=...`.

**What the user sees:** Browser-based SSO flow completes, CLI authenticated.

**Test criteria:**
- [ ] Documentation covers both SSO and static credential paths
- [ ] IAM permission requirements are documented (EKS, VPC, IAM, S3, RDS, OpenSearch, Backup, Secrets Manager)

### Step 4: Store Git Credentials in AWS Secrets Manager

**Action:** Create secret `liferay/credentials/gitops` with keys `git_machine_user_id` and `git_access_token`.

**What the user sees:** Secret created in AWS Console or via CLI.

**Test criteria:**
- [ ] Documentation specifies exact secret name and key names
- [ ] PAT scope requirements are documented (repo read access)
- [ ] Error message is clear if secret is missing or malformed

### Step 5: Create Configuration File

**Action:** Create `config.json` with deployment parameters.

```json
{
  "provider": "aws",
  "variables": {
    "deployment_name": "acme",
    "liferay_git_repo_url": "https://github.com/acme/gitops",
    "liferay_helm_chart_version": "0.1.90",
    "region": "us-west-2"
  }
}
```

**Test criteria:**
- [ ] All required variables are documented
- [ ] Optional variables and their defaults are documented
- [ ] Invalid `deployment_name` (too long, special chars) gives a clear error

### Step 6: Customize Environment Sizing (Optional)

**Action:** Edit `infrastructure.yaml` and `liferay.yaml` per environment.

**What the user configures:**

| File | Knob | Controls |
|------|------|----------|
| `environments/dev/infrastructure.yaml` | `database.instanceClass` | RDS instance size |
| | `database.storageGB` | RDS disk allocation |
| | `search.opensearch.instanceType` | OpenSearch node size |
| | `search.opensearch.instanceCount` | OpenSearch node count |
| `environments/dev/liferay.yaml` | `liferay-default.replicaCount` | Number of Liferay pods |
| | `liferay-default.resources` | CPU/memory per pod |
| `environments/prd/infrastructure.yaml` | Same as dev + `deletionProtection` | Production sizing + safety |

**Test criteria:**
- [ ] Default values work without modification (L-004: instanceCount=1 does NOT work)
- [ ] `deletionProtection: true` works (L-015: currently broken in chart 0.1.90)
- [ ] Documentation explains the base/environment layering model

## Known Issues

| ID | Issue | Severity | Workaround |
|----|-------|----------|------------|
| L-001 | `infrastructure-provider.yaml` missing from boilerplate | Blocks | Create empty file manually |
| L-004 | OpenSearch instanceCount=1 invalid with 2-AZ | Blocks | Set to 2 or 4 |
| L-015 | `deletionProtection: true` breaks Crossplane | Critical | Set to false (loses safety) |

## Future Improvements

- Pre-flight validation script that checks all prerequisites before bootstrap
- Interactive `config.json` generator
- Default `infrastructure-provider.yaml` shipped in boilerplate
- OpenSearch defaults internally consistent (instanceCount matching AZ count)
