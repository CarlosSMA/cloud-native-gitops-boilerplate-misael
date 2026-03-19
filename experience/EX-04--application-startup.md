# EX-04: Application Startup

**Arc:** Get Running
**Persona:** Platform Engineer (monitoring) / Automatic
**Time:** 15-20 minutes per environment (after managed services ready)
**Validation Status:** Validated (with workarounds applied)

## Product Outcome

Liferay DXP starts automatically once managed services are ready. Init containers download OpenSearch modules, seed persistent storage, and configure Tomcat clustering. The main container connects to RDS and OpenSearch using auto-generated credentials. First boot creates the database schema and deploys all OSGi bundles.

## User Journey

### Step 1: Init Containers Run (Automatic)

**Trigger:** `managed-service-details` secret exists in namespace.

**Init container sequence:**

| Order | Container | Purpose | Duration |
|-------|-----------|---------|----------|
| 1 | `liferay-install-opensearch-modules` | Downloads OpenSearch 2 API + Impl JARs | ~10s |
| 2 | `liferay-overlay` | Copies files from S3 overlay bucket (if configured) | ~5s |
| 3 | `liferay-prepopulate-data` | Seeds PV with data dirs, Tomcat cluster config, license | ~5s |
| 4 | `liferay-wait-on-services` | Waits for RDS and OpenSearch to accept connections | varies |

**Test criteria:**
- [ ] All 4 init containers complete successfully
- [ ] OpenSearch modules downloaded for correct product version (L-008/L-013: slim suffix breaks URL)
- [ ] Tomcat `server.xml` includes `CloudMembershipService` for DNS-based clustering
- [ ] License files copied if present

### Step 2: Liferay Main Container Starts (Automatic)

**System behavior:**
- Reads `managed-service-details` secret as environment variables
- Connects to RDS PostgreSQL
- First boot: creates full database schema (~5-10 min)
- Deploys OSGi bundles (~5-10 min)
- Configures OpenSearch connection
- Configures S3 document library

**What the user monitors:**
```bash
kubectl logs liferay-default-0 -n liferay-default-dev -f
# Watch for "Started LiferayGlobalStartupAction" or similar
```

**Test criteria:**
- [ ] Database connection established (check logs for PostgreSQL connection)
- [ ] Schema creation completes without errors
- [ ] OpenSearch connection established
- [ ] S3 document library configured
- [ ] All OSGi bundles resolve (no "Waiting on startup required bundles")

### Step 3: Health Probes Pass (Automatic)

**Probe configuration (from base liferay.yaml):**

| Probe | Path | Period | Threshold | Timeout |
|-------|------|--------|-----------|---------|
| Startup | `/c/portal/robots` | 10s | 90 failures (15 min window) | 5s |
| Readiness | `/c/portal/robots` | 10s | 2 failures | 5s |
| Liveness | `/c/portal/robots` | 30s | 3 failures | 5s |

All probes require `httpHeaders: [{name: Host, value: localhost}]` (L-014).

**What the user sees:**
```bash
kubectl get pods -n liferay-default-dev
# NAME                READY   STATUS    RESTARTS   AGE
# liferay-default-0   1/1     Running   0          18m
```

ArgoCD UI: `default-dev-app` turns green (Synced + Healthy).

**Test criteria:**
- [ ] Pod reaches 1/1 Ready
- [ ] ArgoCD app shows Healthy
- [ ] No crash loops during first boot (startup probe threshold sufficient)
- [ ] `GET /c/portal/robots` returns HTTP 200 with `Host: localhost`
- [ ] `GET /c/portal/robots` returns HTTP 500 with pod IP as Host (expected, L-014)

### Step 4: First Login

**Action:** Retrieve admin password and access Liferay.

```bash
# Get password
kubectl get secret liferay-default -n liferay-default-dev \
  -o jsonpath='{.data.LIFERAY_DEFAULT_PERIOD_ADMIN_PERIOD_PASSWORD}' | base64 -d

# Get NLB endpoint
kubectl get gateway -n liferay-default-dev -o jsonpath='{.items[0].status.addresses[0].value}'
```

**Login:** `test@liferay.com` / (password from above)

**Test criteria:**
- [ ] NLB endpoint resolves and is accessible
- [ ] Login page renders
- [ ] Admin credentials work
- [ ] Control Panel shows RDS as database (not Hypersonic)
- [ ] Search indexes are present (OpenSearch connected)

## Known Issues

| ID | Issue | Severity | Workaround |
|----|-------|----------|------------|
| L-008/L-013 | Slim image tag breaks OpenSearch module URL | Blocks | Override init container with trimSuffix |
| L-010 | Non-slim image ES7 sidecar vs readOnlyRootFilesystem | Blocks | Use slim image |
| L-011 | First boot takes 15+ min (schema creation) | Degrades | Patience; increase startup probe threshold |
| L-014 | Health probes fail without Host:localhost header | Blocks | Override all probes with httpHeaders |
| ISS-C | Pod may start on Hypersonic if secret arrives late | Degrades | Restart pod after secret created |

## Image Tag Decision Tree

```
Want OpenSearch? (yes for AWS)
  |
  +-- Use slim image (2025.q4.12-slim)
  |     +-- Override x-liferay-install-opensearch-modules
  |           with trimSuffix "-slim" (L-013 workaround)
  |
  +-- Use non-slim image (2025.q4.12)
        +-- ES7 sidecar crashes with readOnlyRootFilesystem (L-010)
              +-- Blocked. Must use slim.
```

## Future Improvements

- Helm chart should strip `-slim` suffix in OpenSearch module URL automatically
- Helm chart should include `Host: localhost` in default probe config
- Pod should not schedule until `managed-service-details` secret exists
- First boot progress indicator (beyond raw logs)
- Non-slim image should not ship ES7 sidecar when OpenSearch is configured
