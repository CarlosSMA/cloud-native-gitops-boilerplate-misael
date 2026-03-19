# EX-07: Data Migration

**Arc:** Go Live
**Persona:** Platform Engineer / DevOps
**Time:** 30-60 minutes (depends on data size)
**Validation Status:** Not validated

## Product Outcome

The platform engineer migrates an existing Liferay database and document library into the CNE-managed environment. Database is restored via `psql` through an in-cluster worker pod. Document library is uploaded to S3 via the `aws` CLI. Credentials come from the `managed-service-details` secret that Crossplane already provisioned. After migration, Liferay starts and connects to the imported data.

## Prerequisites

- EX-01 through EX-04 complete (platform running, managed services provisioned)
- Source Liferay database dump available (plain SQL format via `pg_dump`)
- Source document library files available locally
- `aws` CLI configured with S3 access to the target bucket
- `kubectl` configured for the target cluster

## User Journey

### Step 1: Scale Liferay to Zero

**Action:** Edit environment `liferay.yaml`, commit, push.

```yaml
liferay-default:
  replicaCount: 0
```

**Wait:** ArgoCD syncs. All Liferay pods terminate. No application is writing to the database or S3.

**Test criteria:**
- [ ] No Liferay pods running in target namespace
- [ ] ArgoCD shows app synced with 0 replicas

### Step 2: Extract Credentials from managed-service-details

**Action:** Decode the credentials that Crossplane auto-generated.

```bash
NS=liferay-default-prd

# Database
DB_ENDPOINT=$(kubectl get secret managed-service-details -n $NS -o jsonpath='{.data.DATABASE_ENDPOINT}' | base64 -d)
DB_USER=$(kubectl get secret managed-service-details -n $NS -o jsonpath='{.data.DATABASE_USERNAME}' | base64 -d)
DB_PASS=$(kubectl get secret managed-service-details -n $NS -o jsonpath='{.data.DATABASE_PASSWORD}' | base64 -d)
DB_PORT=$(kubectl get secret managed-service-details -n $NS -o jsonpath='{.data.DATABASE_PORT}' | base64 -d)

# S3
S3_BUCKET=$(kubectl get secret managed-service-details -n $NS -o jsonpath='{.data.S3_BUCKET_ID}' | base64 -d)
S3_REGION=$(kubectl get secret managed-service-details -n $NS -o jsonpath='{.data.S3_BUCKET_REGION}' | base64 -d)
```

**Test criteria:**
- [ ] All credential fields are populated (not empty)
- [ ] DATABASE_ENDPOINT resolves to an RDS instance
- [ ] S3_BUCKET_ID exists in AWS

### Step 3: Create Database Dump from Source

**Action:** Export the source Liferay database.

From a docker-compose source:
```bash
docker compose exec -i db bash \
  -c "pg_dump --username liferay lportal" > postgresql_dump.sql
```

From an existing RDS or standalone PostgreSQL:
```bash
pg_dump -h <source-host> -U <source-user> -d lportal > postgresql_dump.sql
```

**Post-processing:** Remove any `\restrict` and `\unrestrict` lines from the dump file.

**Test criteria:**
- [ ] Dump file is valid SQL
- [ ] Dump file size is reasonable for the source database

### Step 4: Prepare Target Database

**Action:** Spin up an adminer pod for inspection (optional) and a postgres worker pod for restore.

```bash
NS=liferay-default-prd

# Spin up postgres worker pod
kubectl run pg-worker -n $NS \
  --image=postgres:16.11 \
  --env="PGPASSWORD=$DB_PASS" \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "pg-worker",
        "image": "postgres:16.11",
        "args": ["sleep", "infinity"],
        "resources": {
          "limits": {"cpu": "1", "memory": "2Gi"},
          "requests": {"cpu": "1", "memory": "2Gi"}
        }
      }]
    }
  }' && \
kubectl wait --for=condition=ready pod/pg-worker -n $NS --timeout=300s
```

**Clean the target database:**
```bash
kubectl exec -n $NS pg-worker -- psql -h $DB_ENDPOINT -U $DB_USER -d postgres -c "DROP DATABASE IF EXISTS lportal;"
kubectl exec -n $NS pg-worker -- psql -h $DB_ENDPOINT -U $DB_USER -d postgres -c "CREATE DATABASE lportal;"
```

**Create the `liferay` role** (required if the source dump references it):
```bash
kubectl exec -n $NS pg-worker -- psql -h $DB_ENDPOINT -U $DB_USER -d lportal -c \
  "CREATE ROLE liferay WITH LOGIN PASSWORD 'placeholder'; GRANT liferay TO $DB_USER;"
```

**Test criteria:**
- [ ] pg-worker pod is Running
- [ ] Can connect to RDS from within the cluster
- [ ] Target database is clean (empty lportal)

### Step 5: Restore Database

**Action:** Copy the dump into the worker pod and execute.

```bash
NS=liferay-default-prd

# Copy dump file
kubectl cp postgresql_dump.sql pg-worker:/tmp/postgresql_dump.sql -n $NS

# Execute restore
kubectl exec -n $NS pg-worker -- \
  psql -h $DB_ENDPOINT -U $DB_USER -d lportal -f /tmp/postgresql_dump.sql
```

**Test criteria:**
- [ ] Restore completes without fatal errors (warnings about roles are acceptable)
- [ ] Table count matches source
- [ ] Key tables exist: `User_`, `Company`, `Group_`, `Layout`, `DLFileEntry`
- [ ] Row counts for key tables match source

### Step 6: Upload Document Library to S3

**Action:** Upload the source document library to the S3 bucket.

```bash
# Empty the target bucket first
aws s3 rm --recursive s3://$S3_BUCKET --region $S3_REGION

# Upload document library
aws s3 cp --recursive ./doclib s3://$S3_BUCKET --region $S3_REGION

# Verify
aws s3 ls --recursive s3://$S3_BUCKET --region $S3_REGION | wc -l
```

For large datasets (100GB+), consider AWS DataSync or S3 Transfer Acceleration:
```bash
aws s3 cp --recursive ./doclib s3://$S3_BUCKET \
  --region $S3_REGION \
  --storage-class STANDARD \
  --only-show-errors
```

**Test criteria:**
- [ ] File count in S3 matches source document library
- [ ] Total size matches source
- [ ] Directory structure preserved (`document_library/` prefix if required)

### Step 7: Scale Liferay Back Up

**Action:** Edit environment `liferay.yaml`, commit, push.

```yaml
liferay-default:
  replicaCount: 1    # or 2 for production
```

**Wait:** ArgoCD syncs. Pod starts with init containers, connects to the restored database.

**Test criteria:**
- [ ] Pod reaches 1/1 Ready
- [ ] Liferay connects to restored database (not Hypersonic)
- [ ] Liferay connects to S3 document library
- [ ] Login works with migrated user credentials
- [ ] Documents are accessible in the Liferay UI
- [ ] Search indexes rebuild from restored data

### Step 8: Cleanup

**Action:** Remove temporary pods.

```bash
kubectl delete pod pg-worker -n $NS
```

**Test criteria:**
- [ ] No temporary pods remain
- [ ] ArgoCD shows app Healthy

## Blue/Green Slot Awareness

The `managed-service-details` secret points to whichever slot is `activeSlot` (blue by default). During initial migration, you restore into the active slot directly. The standby slot remains empty and available for future DR restores.

For subsequent migrations or DR, the Argo Workflows restore DAG (EX-12) restores to the standby slot and promotes on success.

## CLI Tools Summary

| Tool | Purpose | Where |
|------|---------|-------|
| `kubectl` | Pod operations, secret reads, file copy | Operator workstation |
| `aws s3` | Upload document library to S3 bucket | Operator workstation |
| `psql` | Execute SQL dump restore | Inside `postgres:16.11` pod (in-cluster) |
| `pg_dump` | Create source database dump | Source environment |
| `base64` | Decode secret values | Operator workstation |

## Known Issues

| Issue | Severity | Notes |
|-------|----------|-------|
| Source dump may reference `liferay` role not present in target | Degrades | Create role manually before restore |
| `pg_dump` default format includes `\restrict` lines | Degrades | Strip from dump file |
| Large doclib uploads may timeout | Degrades | Use `--only-show-errors` and multipart config |
| No automated migration workflow | Product gap (R-08) | Entire process is manual |

## Future Improvements

- Automated migration workflow via Argo Workflows (R-08)
- Migration validation step (row count comparison, integrity check)
- Progress indicator for large S3 uploads
- Support for non-PostgreSQL source databases (MySQL migration path)
- Single CLI command for the full migration sequence
