# Cloud Native Experience Map

This directory maps every user touchpoint with the Liferay Cloud Native Operating Kit, organized around the product's three pillars: **Provision Infrastructure**, **Deploy and Update Application**, and **Operate Environments**.

## Purpose

1. **Describe the product end-to-end** from the user's perspective
2. **Define a test plan** with expected outcomes for each experience
3. **Plan for the future** by identifying gaps, friction, and roadmap alignment

Canonical feature reference: [`/home/misael/pj/cn-value-prop2/output/operating-kit.md`](/home/misael/pj/cn-value-prop2/output/operating-kit.md)

---

## Provision Infrastructure

One command provisions the complete platform: Kubernetes cluster, networking, GitOps engine, secrets operator, backup automation, ingress gateway.

| # | Experience | Persona | Time | Status |
|---|-----------|---------|------|--------|
| EX-01 | [Prerequisites and Setup](EX-01--prerequisites-and-setup.md) | Platform Engineer | 30 min | Validated |
| EX-02 | [Platform Provisioning](EX-02--platform-provisioning.md) | Platform Engineer | 20-40 min | Validated |
| EX-03 | [Managed Services](EX-03--managed-services.md) | Platform Engineer | 20-45 min | Validated |
| EX-04 | [Application Startup](EX-04--application-startup.md) | Platform Engineer | 15-20 min | Validated |

The bootstrap script outputs the ArgoCD admin password and port-forward command at the end of EX-02. ArgoCD is accessible from that point forward.

**Total time to running Liferay: 45-70 minutes.**

---

## Deploy and Update Application

Production-ready Helm chart with clustering, health monitoring, hardened defaults, and managed service integrations. Two deployment workflows, PR-based promotion, and drift correction.

| # | Experience | Persona | Time | Status |
|---|-----------|---------|------|--------|
| EX-05 | [DNS and TLS](EX-05--dns-and-tls.md) | Platform Engineer | 15 min | Not validated |
| EX-06 | [License Activation](EX-06--license-activation.md) | Platform Engineer | 5 min | Not validated |
| EX-07 | [Data Migration](EX-07--data-migration.md) | Platform Engineer / DevOps | 30-60 min | Not validated |
| EX-08 | [Scaling](EX-08--scaling.md) | DevOps | 5 min | Not validated |
| EX-09 | [Version Upgrades](EX-09--version-upgrades.md) | DevOps | 10 min | Not validated |
| EX-10 | [Adding Environments](EX-10--adding-environments.md) | DevOps / Platform Engineer | 5 min + 45 min provision | Not validated |
| EX-11 | [Application Deployments](EX-11--application-deployments.md) | Developer / DevOps | varies | Not validated |

---

## Operate Environments

ArgoCD console, CloudWatch baseline, automated backups, autoscaling, secrets sync, and drift correction.

| # | Experience | Persona | Time | Status |
|---|-----------|---------|------|--------|
| EX-12 | [Backup and Restore](EX-12--backup-and-restore.md) | Platform Engineer | varies | Not validated |
| EX-13 | [Observability](EX-13--observability.md) | DevOps / Platform Engineer | ongoing | Partial |
| EX-14 | [Drift Detection and Security](EX-14--drift-and-security.md) | Security / Platform Engineer | ongoing | Implicit |
| EX-15 | [Teardown](EX-15--teardown.md) | Platform Engineer | 15-30 min | Not validated |

---

## Cross-Cutting Concerns

| Concern | Experiences Affected | Notes |
|---------|---------------------|-------|
| GitOps loop | EX-08 through EX-14 | Every change is a Git commit. ArgoCD reconciles. |
| IRSA (workload identity) | EX-02, EX-03, EX-11, EX-13 | No static credentials anywhere. |
| Crossplane reconciliation | EX-03, EX-08, EX-10, EX-12 | Infrastructure as CRDs, not Terraform after bootstrap. |
| Helm values layering | EX-04, EX-08, EX-09, EX-11 | base/ + environments/*/ merged by ArgoCD. |

## Known Issues Summary

| ID | Experience | Severity | Type | Description |
|----|-----------|----------|------|-------------|
| L-001 | EX-01 | Blocks | Doc gap | Missing `infrastructure-provider.yaml` in boilerplate |
| L-004 | EX-03 | Blocks | Product gap | OpenSearch instanceCount=1 invalid with 2-AZ defaults |
| L-008/L-013 | EX-04 | Blocks | Product gap | Slim image tag breaks OpenSearch module URL |
| L-010 | EX-04 | Blocks | Product gap | Non-slim image ES7 sidecar vs readOnlyRootFilesystem |
| L-014 | EX-04 | Blocks | Product gap | Health probes fail without Host:localhost header |
| L-015 | EX-03 | Critical | Product bug | deletionProtection breaks Crossplane reference resolution |
| ISS-A | EX-03 | Blocks | Product gap | S3 naming collision on generic project names |

## Config Surface Map

Every file the user touches, mapped to which experience it belongs to:

| File | Experiences | Frequency |
|------|------------|-----------|
| `config.json` | EX-01 | Once |
| `liferay/system/infrastructure-provider.yaml` | EX-01 | Once |
| `liferay/projects/*/base/liferay.yaml` | EX-04, EX-09, EX-11 | Per upgrade |
| `liferay/projects/*/environments/*/liferay.yaml` | EX-08, EX-11 | Per scaling/deploy |
| `liferay/projects/*/environments/*/infrastructure.yaml` | EX-03, EX-05, EX-06, EX-08 | Per infra change |
