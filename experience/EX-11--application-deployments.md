# EX-11: Application Deployments

**Arc:** Operate
**Persona:** Developer / DevOps
**Time:** Varies by workflow
**Validation Status:** Not validated

## Product Outcome

Three deployment workflows serve different use cases. All use Git as the source of truth with full audit trail. Deployments promote through environments via PR. Every change has an author, timestamp, and rollback path.

## Deployment Workflows

### Workflow A: Custom Image Deploy

**Use case:** Full application build with customizations baked into the container image.

**User journey:**
1. Developer builds custom image in CI (GitHub Actions, Jenkins, etc.)
2. CI pushes image to container registry (ECR with immutable tags + scan-on-push)
3. DevOps opens PR updating `base/liferay.yaml` image tag
4. PR review and merge
5. ArgoCD syncs. Rolling update deploys new image.

**Test criteria:**
- [ ] CI can push to ECR using provisioned CI credentials
- [ ] ECR rejects mutable tag overwrites
- [ ] Scan-on-push triggers CVE scan
- [ ] PR merge triggers ArgoCD sync
- [ ] Rolling update completes without downtime

### Workflow B: Bundle Deploy (S3 Overlay)

**Use case:** Configuration changes, OSGi modules, client extensions without rebuilding the image.

**User journey:**
1. CI builds Gradle workspace
2. CI uploads artifacts to S3 overlay bucket (using CI uploader credentials)
3. DevOps edits `liferay.yaml` to reference the overlay build
4. Commit, push. ArgoCD syncs.
5. Pod restarts. `liferay-overlay` init container copies files from S3.

```yaml
liferay-default:
  overlay:
    enabled: true
    bucketName: <overlay-bucket-name>
    copy:
      - from: "build_42/osgi/client-extensions/*"
        into: osgi/client-extensions
```

**Retrieving CI credentials:**
```bash
kubectl get secret <overlay-bucket>-ci-uploader-access-key -n <namespace> \
  -o jsonpath='{.data.username}' | base64 -d
kubectl get secret <overlay-bucket>-ci-uploader-access-key -n <namespace> \
  -o jsonpath='{.data.password}' | base64 -d
```

**Test criteria:**
- [ ] CI credentials are append-only (explicit deny on delete)
- [ ] S3 upload succeeds
- [ ] Init container copies files to correct paths
- [ ] Liferay picks up deployed modules on startup
- [ ] Rollback: change overlay reference in Git

### Workflow C: Hot-Deploy (Roadmap)

**Use case:** Deploy modules to a running container without restart.

**Status:** Not shipped. On roadmap (R-04).

## Environment Promotion

**Pattern:** dev -> uat -> prd via PR.

1. Test change in dev (merge PR to dev liferay.yaml or base)
2. Open PR promoting same change to prd (edit prd liferay.yaml)
3. Review, approve, merge
4. ArgoCD syncs prd

**Test criteria:**
- [ ] Infrastructure and application PRs are independent (separate approval workflows)
- [ ] Rollback: `git revert` the promotion PR

## Client Extensions

Client Extensions deploy via the same workflows (A or B). No special handling required beyond the standard GitOps flow.

**Test criteria:**
- [ ] Client Extension JARs deploy via overlay
- [ ] Client Extensions activate in Liferay without errors

## Future Improvements

- Hot-deploy without restart (R-04)
- PR preview environments (ephemeral)
- Deployment status notifications (Slack, email)
- Automated smoke tests post-deploy
