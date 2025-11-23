# Cert-Manager + NGINX Gateway (Let's Encrypt)

This folder contains configuration and guidance for issuing TLS certificates from Let's Encrypt using cert-manager integrated with the Kubernetes Gateway API (NGINX implementation).

## 1. Purpose
Enable automatic HTTP/HTTPS certificate management for wildcard and specific hostnames served by the `nginx-gateway` using ACME challenges (HTTP-01 and DNS-01). DNS-01 supports wildcard (`*.codeify.me`).

## 2. Files
| File | Description |
|------|-------------|
| `certmanager-nginxgateway-issuer.yaml` | ACME Issuer with HTTP-01 + DNS-01 solvers |

## 3. Prerequisites
- Kubernetes cluster
- Gateway API CRDs installed
- NGINX Gateway controller deployed (gatewayClass `nginx`)
- Domain DNS records pointing to the Gateway LoadBalancer
- DNS provider token secret (for DNS-01) created in `nginx-gateway` namespace (e.g. `digitalocean-dns`)

## 4. Enable Gateway API in cert-manager
Gateway API support must be enabled for cert-manager to use HTTP-01 challenges via Gateway resources. This is configured via Helm values or CLI flags.

### 4.1 Helm Values Approach
Add these values to your Helm chart:
```yaml
config:
  apiVersion: controller.config.cert-manager.io/v1alpha1
  kind: ControllerConfiguration
  enableGatewayAPI: true
```

### 4.2 Helm Install Command
Or apply directly during installation:
```bash
helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager --namespace cert-manager \
  --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
  --set config.kind="ControllerConfiguration" \
  --set config.enableGatewayAPI=true
```

### 4.3 Important: Restart Required
Cert-manager components only check the Gateway API flag on startup. If you enable it after initial installation, restart the deployment:
```bash
kubectl rollout restart deployment cert-manager -n cert-manager
```

## 5. Issuer Overview (`certmanager-nginxgateway-issuer.yaml`)
Key sections:
- `spec.acme.server`: Production Let's Encrypt directory
- `profile: tlsserver`: Certificate profile
- `privateKeySecretRef`: Stores ACME account key
- `solvers`:
  - `http01.gatewayHTTPRoute`: Solves via Gateway HTTPRoute (good for single hosts)
  - `dns01.digitalocean`: Solves via DNS records (required for wildcard)

```yaml
solvers:
  - http01:
      gatewayHTTPRoute:
        parentRefs:
          - name: nginx-gateway
            namespace: nginx-gateway
  - dns01:
      digitalocean:
        tokenSecretRef:
          name: digitalocean-dns
          key: access-token
```

## 6. Wildcard vs Single Host
| Use Case | Recommended Solver | Notes |
|----------|--------------------| ------|
| `*.codeify.me` wildcard | DNS-01 | **Only DNS-01 can issue wildcard certificates** |
| `app.codeify.me` only | HTTP-01 or DNS-01 | HTTP verification works for single hosts |

### 6.1 HTTP-01 Challenge Issues
Currently, HTTP-01 challenges may encounter routing or visibility problems depending on Gateway configuration. If HTTP-01 fails, use DNS-01 as the reliable alternative.

**Important**: Wildcard domain certificates (`*.example.com`) can **only** be issued via DNS-01 validation. HTTP verification does not support wildcard patterns.

### 6.2 Documentation References
- [cert-manager HTTP-01 Challenge](https://cert-manager.io/docs/configuration/acme/http01/)
- [cert-manager Gateway API Usage](https://cert-manager.io/docs/usage/gateway/)

## 7. Gateway Annotation
In `nginx-gateway.yaml`, ensure:
```yaml
metadata:
  annotations:
    cert-manager.io/issuer: letsencrypt-codeify
```
This drives certificate creation (Certificate resource -> Secret referenced by Gateway listener `certificateRefs`).

## 8. Apply Sequence
```bash
# 1. Install cert-manager (with Gateway API enabled)
#    (helm command from section 4)

# 2. Create Issuer
kubectl apply -f nginx-fabric/cert-manager-letsencrypt/certmanager-nginxgateway-issuer.yaml

# 3. Deploy Gateway (if not already)
kubectl apply -f nginx-fabric/nginx-gateway.yaml

# 4. Check certificate issuance
kubectl get certificate -n nginx-gateway
kubectl describe certificate -n nginx-gateway
```


## 9. Troubleshooting
| Symptom | Check | Action |
|---------|-------|--------|
| Issuer not Ready | `kubectl describe issuer -n <namespace>` | Fix DNS token / email / server URL |
| Wildcard/http fails | `kubectl get orders.acme.cert-manager.io -n <namespace>` | Confirm DNS-01 solver + token secret |
| HTTP-01 pending | HTTPRoute challenge path | Ensure Gateway API feature enabled + route reachable |
| Secret missing | `kubectl get certificates.cert-manager.io -A` | Reconcile cert-manager pods / verify solver |
| Certificate not Ready | `kubectl get certificaterequests.cert-manager.io -A` | Check APPROVED/READY columns and issuer reference |

### 9.1 Diagnostic Commands
```bash
# View all certificates across namespaces
kubectl get certificates.cert-manager.io -A

# Check certificate requests (shows approval status)
kubectl get certificaterequests.cert-manager.io -A

# Inspect ACME orders for challenge details
kubectl get orders.acme.cert-manager.io -n <namespace>

# Describe specific certificate for events
kubectl describe certificate <cert-name> -n <namespace>
```

**Example Output** (successful issuance):
```
NAMESPACE       NAME               READY   SECRET             AGE
nginx-gateway   codeify-me-tls     True    codeify-me-tls     18h
```

## 10. Next Steps
- Switch to `ClusterIssuer` if multi-namespace cert management broadens.
- Add staging Issuer for testing (`https://acme-staging-v02.api.letsencrypt.org/directory`).
- Integrate automated renewal monitoring via events or Prometheus metrics.

---
