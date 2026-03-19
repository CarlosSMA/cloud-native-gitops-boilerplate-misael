# EX-06: License Activation

**Arc:** Go Live
**Persona:** Platform Engineer
**Time:** ~5 minutes
**Validation Status:** Not validated

## Product Outcome

The platform engineer uploads a Liferay license to AWS Secrets Manager. ESO syncs it to Kubernetes. The license is mounted into Liferay pods via the Helm chart's volume system. No manual file copying or pod exec required.

## User Journey

### Step 1: Store License in AWS Secrets Manager

**Action:** Upload license XML to Secrets Manager.

- Secret name: `liferay/licenses/<name>` (prefix required)
- Key: `license.xml` (base64 -w 0 encoded XML content)

### Step 2: Configure License in infrastructure.yaml

**Action:** Edit `infrastructure.yaml`, commit, push.

```yaml
license:
  externalSecretName: "liferay/licenses/dxp-license"
  targetSecretName: "liferay-license"
```

### Step 3: Mount License in liferay.yaml

**Action:** Edit `liferay.yaml`, commit, push.

```yaml
liferay-default:
  customEnv:
    x-license:
      - name: LIFERAY_DISABLE_TRIAL_LICENSE
        value: "true"
  customVolumeMounts:
    x-license:
      - mountPath: /etc/liferay/mount/files/deploy
        name: license
  customVolumes:
    x-license:
      - name: license
        secret:
          secretName: liferay-license
          items:
            - key: license.xml
              path: license.xml
```

**System response:** ArgoCD syncs. Pod restarts. Init container copies license to deploy directory. Liferay starts with the production license.

**Test criteria:**
- [ ] ExternalSecret syncs license from Secrets Manager
- [ ] License secret exists in namespace
- [ ] Pod mounts license file at correct path
- [ ] Liferay Control Panel shows production license (not trial)
- [ ] License renewal: update Secrets Manager, ESO syncs within 1 hour, pod restart picks up new license

## Future Improvements

- License validation at sync time (is the XML well-formed?)
- License expiry monitoring
