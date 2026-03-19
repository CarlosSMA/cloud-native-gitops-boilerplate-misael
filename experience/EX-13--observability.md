# EX-13: Observability

**Arc:** Operate
**Persona:** DevOps / Platform Engineer
**Time:** Ongoing
**Validation Status:** Partial (CloudWatch deployed, Grafana stack not shipped)

## Product Outcome

Out-of-the-box cluster and container metrics via CloudWatch. JMX Prometheus agent pre-wired in the Liferay image for application-level metrics. Future: pre-built Grafana dashboards for Liferay-specific health signals (JVM, DB pools, search latency, document store throughput).

## What Ships Today

### CloudWatch Container Insights (Automatic)

**No user action required.** Installed during EKS bootstrap.

**What the user sees:**
- AWS Console -> CloudWatch -> Container Insights
- Cluster-level: CPU, memory, network, disk per node
- Pod-level: CPU, memory, restarts, status
- Container-level: resource utilization vs limits
- Logs: all pod stdout/stderr in `/aws/eks/<cluster-name>/cluster`
- Retention: 90 days

**Test criteria:**
- [ ] Container Insights shows cluster metrics
- [ ] Pod metrics visible for Liferay pods
- [ ] Logs searchable in CloudWatch Logs
- [ ] 90-day retention confirmed

### Kubernetes Events

**Available via:** `kubectl get events -n <namespace>` or CloudWatch logs.

### JMX Prometheus Agent (Pre-wired, Not Consumed)

The Liferay image includes a JMX Prometheus exporter (port 9404). It exposes JVM metrics (heap, GC, threads) but nothing consumes them by default.

**Optional activation via `liferay.yaml`:**
```yaml
liferay-default:
  customEnv:
    x-jmx:
      - name: JMX_AGENT_YAML
        value: |
          rules:
            - pattern: "java.lang<type=Memory><>(HeapMemoryUsage|NonHeapMemoryUsage)"
      - name: JMX_AGENT_PORT
        value: "9090"
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

**Test criteria:**
- [ ] JMX agent starts on configured port
- [ ] `/metrics` endpoint returns Prometheus-format metrics
- [ ] Customer's own Prometheus can scrape the endpoint

## What Doesn't Ship Today (Roadmap)

| Feature | Roadmap ID | Gap |
|---------|-----------|-----|
| Grafana Alloy (DaemonSet metrics collector) | R-06 | No automated Liferay-specific metric collection |
| Amazon Managed Prometheus (AMP) | R-07 | No managed metrics backend |
| Pre-built Grafana dashboards | R-08 | No JVM, DB pool, search latency dashboards |
| Unified metric names across providers | R-09 | No cross-cloud portability |
| Grafana-based alerting | R-10 | No alerting on Liferay health signals |
| FinOps labels | R-11 | No cost attribution per environment/workload |

## Debugging Commands

```bash
# Pod status
kubectl get pods -n liferay-default-dev -o wide

# Pod logs (current)
kubectl logs liferay-default-0 -n liferay-default-dev -f

# Pod logs (previous crash)
kubectl logs liferay-default-0 -n liferay-default-dev --previous

# Init container logs
kubectl logs liferay-default-0 -c liferay-install-opensearch-modules -n liferay-default-dev

# Resource utilization
kubectl top pods -n liferay-default-dev
kubectl top nodes

# Crossplane resource status
kubectl get managed -A | grep -v True

# Envoy Gateway status
kubectl get gateway -A
kubectl get httproute -A
```

## Future Improvements

- Pre-built Grafana dashboards for Liferay JVM health, database connection pools, search latency
- Managed Prometheus (AMP) with 15-day metric retention
- Automated alerting on Liferay-specific signals
- FinOps labels for cost drill-down by environment
