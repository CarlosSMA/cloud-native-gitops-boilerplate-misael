# EX-05: DNS and TLS

**Arc:** Go Live
**Persona:** Platform Engineer
**Time:** ~15 minutes (plus DNS propagation)
**Validation Status:** Not validated

## Product Outcome

The platform engineer configures a custom domain with TLS for Liferay. Certificates are stored in AWS Secrets Manager and synced hourly to Kubernetes via External Secrets Operator. Envoy Gateway handles TLS termination and HTTP-to-HTTPS redirect. No manual certificate rotation required.

## User Journey

### Step 1: Store TLS Certificate in AWS Secrets Manager

**Action:** Upload cert and key to Secrets Manager.

- Secret name: `liferay/certificates/<name>` (prefix required by ESO scope)
- Keys: `tls.crt` (base64 -w 0 of PEM cert), `tls.key` (base64 -w 0 of PEM key)

**Test criteria:**
- [ ] Documentation specifies exact encoding (base64 -w 0, not raw PEM)
- [ ] Secret name prefix `liferay/certificates/` is enforced and documented
- [ ] Error message is clear if cert format is wrong

### Step 2: Configure Endpoint in infrastructure.yaml

**Action:** Edit `infrastructure.yaml`, commit, push.

```yaml
endpoints:
  mydomain:
    hostname: "app.example.com"
    port: 443
    protocol: HTTPS
    tls:
      externalSecretNames:
        - "liferay/certificates/example-tls"
```

**System response:** ArgoCD syncs. Crossplane creates an ExternalSecret that pulls the cert from Secrets Manager. A Kubernetes TLS Secret is created. The environment's Gateway gets an HTTPS listener.

**Test criteria:**
- [ ] ExternalSecret is created and synced (SecretSynced condition)
- [ ] TLS Secret exists in environment namespace
- [ ] Gateway shows HTTPS listener on port 443
- [ ] NLB is created with correct listeners

### Step 3: Create DNS CNAME Record

**Action:** Get the NLB DNS name and create a CNAME in DNS provider.

```bash
kubectl get gateway -n liferay-default-prd -o jsonpath='{.items[0].status.addresses[0].value}'
```

**Test criteria:**
- [ ] NLB DNS name is populated in Gateway status
- [ ] DNS CNAME resolves to NLB
- [ ] HTTPS connection succeeds with valid cert
- [ ] HTTP-to-HTTPS redirect works (301)

### Step 4: Enable Network Routing in liferay.yaml

**Action:** Edit `liferay.yaml`, commit, push.

```yaml
liferay-default:
  network:
    enabled: true
    endpointRef: mydomain
    forceHttpsRedirect: true
    hostnames:
      - "app.example.com"
```

**System response:** ArgoCD syncs. HTTPRoute binds to the HTTPS listener. Traffic flows: Internet -> NLB (L4) -> Envoy (L7, TLS termination) -> Liferay pods (:8080).

**Test criteria:**
- [ ] HTTPRoute is created and accepted by Gateway
- [ ] `curl -I https://app.example.com` returns HTTP 200
- [ ] `curl -I http://app.example.com` returns HTTP 301 to HTTPS
- [ ] Liferay login page loads via domain

## Certificate Renewal

**User action:** Upload new cert to same Secrets Manager path. No other action needed.

**System response:** ESO syncs new cert within 1 hour (configurable refresh interval). Envoy picks up new TLS Secret automatically.

**Test criteria:**
- [ ] New cert is picked up without pod restart
- [ ] No downtime during cert rotation
- [ ] Old cert is replaced, not appended

## Known Issues

| ID | Issue | Severity | Workaround |
|----|-------|----------|------------|
| R-03 | No automatic TLS via cert-manager/Let's Encrypt | Product gap | BYO cert via Secrets Manager |

## Future Improvements

- cert-manager integration for automatic Let's Encrypt certificates
- Validation that cert matches configured hostname
- Cert expiry monitoring and alerting
