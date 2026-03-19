# EX-08: Scaling

**Arc:** Operate
**Persona:** DevOps
**Time:** ~5 minutes (Git change) + auto-reconciliation
**Validation Status:** Not validated

## Product Outcome

Scaling Liferay is a Git PR. Change the replica count or resource limits in `liferay.yaml`, merge, and ArgoCD reconciles. Horizontal pod autoscaler (HPA) handles automatic scaling within configured bounds. JGroups clustering activates automatically when replicas > 1. No JGroups XML, S3_PING, or TCP_PING configuration needed.

## User Journey

### Horizontal Scaling (Manual)

**Action:** Edit environment `liferay.yaml`, commit, push.

```yaml
liferay-default:
  replicaCount: 3    # was 2
```

**System response:** ArgoCD syncs. StatefulSet scales to 3. New pod starts with same init container sequence. Tomcat DNS-based `CloudMembershipService` discovers the new member automatically.

**Test criteria:**
- [ ] New pod starts and reaches Ready
- [ ] JGroups cluster forms (check logs for membership change)
- [ ] Session replication works across all pods
- [ ] No manual JGroups configuration required
- [ ] Scale-down removes pods in reverse ordinal order

### Vertical Scaling

**Action:** Edit environment `liferay.yaml`, commit, push.

```yaml
liferay-default:
  resources:
    requests:
      cpu: "12"
      memory: 12Gi
    limits:
      cpu: "12"
      memory: 12Gi
```

**System response:** ArgoCD syncs. StatefulSet triggers rolling update. Pods restart one at a time with new resource allocation.

**Test criteria:**
- [ ] Rolling update completes without downtime (if replicas > 1)
- [ ] New resource limits reflected in pod spec
- [ ] Node autoscaling triggers if nodes lack capacity

### Autoscaling (HPA)

**Action:** Configure HPA via `liferay.yaml`.

```yaml
liferay-default:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    targetCPUUtilizationPercentage: 75
```

**System response:** HPA scales pods between 2-5 based on CPU utilization.

**Test criteria:**
- [ ] HPA created and active
- [ ] Scale-out triggers at >75% CPU
- [ ] Scale-in occurs when load drops
- [ ] Min replicas respected

### Infrastructure Scaling

**Action:** Edit environment `infrastructure.yaml`, commit, push.

```yaml
database:
  instanceClass: db.r6g.2xlarge    # was db.r6g.xlarge
  storageGB: 100                    # was 50
```

**System response:** ArgoCD syncs. Crossplane modifies RDS instance. AWS applies change (may cause brief maintenance window).

**Test criteria:**
- [ ] RDS instance class changes without data loss
- [ ] Storage expansion completes
- [ ] No Terraform re-run required

## Future Improvements

- KEDA autoscaling on Liferay-specific signals (R-05): session counts, thread pools, request queues
- Scaling recommendations based on observed metrics
