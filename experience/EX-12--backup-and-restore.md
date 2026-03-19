# EX-12: Backup and Restore

**Arc:** Operate
**Persona:** Platform Engineer
**Time:** Varies (backup config: 5 min; restore execution: 30-60 min)
**Validation Status:** Not validated

## Product Outcome

Automated backups via AWS Backup with tag-based resource selection. Disaster recovery via Argo Workflows with a nine-step DAG that restores to the standby (blue/green) plane, validates, and promotes on success. Auto-rollback on failure.

## User Journey

### Configure Backups

**Action:** Edit `infrastructure.yaml`, commit, push.

```yaml
backup:
  enabled: true
  rules:
    - ruleName: hourly-backups
      schedule: "cron(0 * * * ? *)"
      retentionDays: 30
      startWindow: 60
```

**System response:** Crossplane provisions AWS Backup plan covering RDS and S3 in this environment. Backups run on schedule.

**Test criteria:**
- [ ] AWS Backup plan created
- [ ] Backup executes on schedule
- [ ] Recovery points are listed in AWS Backup vault
- [ ] Tags correctly identify environment resources

### Execute Restore

**Action:** Access Argo Workflows UI and submit restore workflow.

```bash
kubectl port-forward -n argo-workflows-system svc/argo-workflows-server 2746:2746
# Browser: http://localhost:2746
```

Submit workflow:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: backup-restore-
  namespace: liferay-default-dev
spec:
  workflowTemplateRef:
    clusterScope: true
    name: backup-restore-cluster-workflow-template
  arguments:
    parameters:
      - name: recovery-point-arn
        value: "arn:aws:rds:us-east-2:...:snapshot:..."
```

**System response:** Nine-step DAG:
1. Scale Liferay to 0
2. Restore RDS to standby slot
3. Restore S3 to standby slot
4. Validate restored data
5. Switch active slot pointer
6. Update `managed-service-details` secret
7. Scale Liferay back up
8. Verify Liferay connects to restored data
9. Auto-rollback on any failure

**Test criteria:**
- [ ] Argo Workflows UI accessible
- [ ] Restore workflow completes successfully
- [ ] Data integrity verified post-restore
- [ ] Active slot switches correctly
- [ ] Auto-rollback works on simulated failure
- [ ] No data loss

### Blue/Green Slot Architecture

The composition maintains `blue` and `green` RDS and S3 slots. `activeSlot` in `infrastructure.yaml` controls which slot Liferay connects to.

**Test criteria:**
- [ ] Both slots exist in AWS
- [ ] Switching `activeSlot` re-templates `managed-service-details`
- [ ] Liferay reconnects to new slot on pod restart

## Future Improvements

- One-click restore from ArgoCD UI
- Restore progress indicator
- Automated restore testing on schedule
- Migration from external sources (R-08)
