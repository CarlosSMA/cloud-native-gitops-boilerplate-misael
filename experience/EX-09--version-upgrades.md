# EX-09: Version Upgrades

**Arc:** Operate
**Persona:** DevOps
**Time:** ~10 minutes (Git change) + rolling update
**Validation Status:** Not validated

## Product Outcome

Upgrading Liferay DXP is a single image tag change in Git. ArgoCD performs a rolling update. Init containers re-download OpenSearch modules for the new version. The Operating Kit also receives security updates from Liferay via Helm chart version bumps.

## User Journey

### DXP Version Upgrade

**Action:** Edit `base/liferay.yaml`, commit, push.

```yaml
liferay-default:
  image:
    tag: 2025.q4.13-slim    # was 2025.q4.12-slim
```

**System response:** ArgoCD syncs. StatefulSet triggers rolling update across all environments that inherit from base. Init containers re-run. `liferay-install-opensearch-modules` downloads JARs for the new version.

**Test criteria:**
- [ ] Rolling update completes without downtime (replicas > 1)
- [ ] OpenSearch modules re-downloaded for correct version
- [ ] Database migration scripts run (if any for this version)
- [ ] All OSGi bundles resolve after upgrade
- [ ] Rollback: `git revert` the PR restores previous version

### Per-Environment Version Pinning

**Action:** Override image tag in environment `liferay.yaml` (e.g., pin dev to a newer version for testing).

```yaml
liferay-default:
  image:
    tag: 2025.q4.14-slim    # testing ahead of production
```

**Test criteria:**
- [ ] Only the target environment gets the new version
- [ ] Base tag still applies to other environments

### Operating Kit (Helm Chart) Upgrade

**Action:** Update chart version in ArgoCD ApplicationSet (Terraform variable or direct edit).

```
targetRevision: "0.1.91"    # was 0.1.90
```

**Test criteria:**
- [ ] New chart version deploys without breaking existing config
- [ ] Custom overrides (probes, init containers) still apply
- [ ] Security patches take effect

## Future Improvements

- Canary deployment support (route % of traffic to new version)
- Automated smoke tests after upgrade
- Version compatibility matrix in documentation
