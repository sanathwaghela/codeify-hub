# Cross-Namespace Routing (Gateway API)

This folder documents how HTTPRoutes in different namespaces can attach to a single NGINX Gateway using Kubernetes Gateway API, enabling multi-tenant ingress with controlled access.

## 1. Purpose
Allow services deployed across multiple namespaces (e.g. `application`, `route-app`) to expose hostnames via one shared `Gateway` while enforcing namespace-level access control.

## 2. Files
| File | Description |
|------|-------------|
| `cross-namespace-route.yaml` | Two example `HTTPRoute` resources mapping hostnames to backend services |

## 3. Core Concepts
- `Gateway` lives in namespace `nginx-gateway` and defines listeners (HTTP/HTTPS) + hostname patterns.
- `HTTPRoute` resources in other namespaces reference the Gateway via `parentRefs`.
- Access controlled by `Gateway.spec.listeners.allowedRoutes.namespaces` policy (selector-based here).

## 3.1 General Overview
- An `nginx-gateway` (kind: `Gateway`) creates a cloud LoadBalancer when the controller supports it.
- A single gateway can serve multiple namespaces via attached `HTTPRoute` (and other route types like `TCPRoute`) objects.

## 3.2 Cross-Namespace Access Models
Two patterns for `allowedRoutes.namespaces`:
```yaml
allowedRoutes:
  namespaces:
    from: All        # Any namespace may attach routes (least restrictive)
```
OR selector-based restriction:
```yaml
allowedRoutes:
  namespaces:
    from: Selector
    selector:
      matchLabels:
        gateway-access: "true"
```
Selector mode is recommended for multi-tenant security; only labeled namespaces can attach.

Namespace label example:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: application
  labels:
    gateway-access: "true"
spec: {}
```

## 3.3 Attaching Routes (ParentRefs)
Each route must explicitly reference the gateway namespace, name, and optional listener section:
```yaml
spec:
  parentRefs:
    - name: nginx-gateway      # Gateway name
      sectionName: codeify-https # Listener name defined on the Gateway
      namespace: nginx-gateway   # Namespace where Gateway resides
```

## 3.4 Why `from: All` Works
Using `from: All` disables namespace filtering entirely so any route from any namespace is accepted. This is useful for quick demos but not advised for production isolation.

## 3.5 Troubleshooting Acceptance
Describe a route to inspect attachment:
```bash
kubectl describe httproute -n application <route-name>
```
Look for:
```
Parent Refs:
  Group:         gateway.networking.k8s.io
  Kind:          Gateway
  Name:          nginx-gateway
  Namespace:     nginx-gateway
  Section Name:  codeify-https
```
If the namespace lacks the `gateway-access: "true"` label under selector mode, the route will be Rejected.

## 3.6 Common Rejection Causes
- Missing namespace label when selector mode is active.
- Wrong `sectionName` (listener name mismatch).
- Typo in `namespace` under `parentRefs`.
- Gateway not Ready yet (check `kubectl get gateway -n nginx-gateway`).

## 3.7 Quick Diagnostic Commands
```bash
kubectl -n application get httproutes.gateway.networking.k8s.io
kubectl describe httproute application-route -n application
kubectl describe httproute route-app -n route-app
```

---

## 4. Namespace Eligibility
Two allowed strategies:
1. `from: All` – any namespace may attach routes (simple, less secure).
2. `from: Selector` – only namespaces with matching labels (recommended).

Example label requirement:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: application
  labels:
    gateway-access: "true"
```
Both `application` and `route-app` namespaces include this label.

## 5. HTTPRoute Parent References
Each route must specify:
```yaml
parentRefs:
  - name: nginx-gateway      # Gateway name
    sectionName: codeify-https # Listener name (port/protocol defined in Gateway)
    namespace: nginx-gateway  # Namespace where Gateway resides
```
`sectionName` binds to the listener; using HTTPS listener ensures TLS termination handled at Gateway.

### 5.1 Gateway Manifest (Reference)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: nginx-gateway
  annotations:
    cert-manager.io/issuer: letsencrypt-codeify
spec:
  gatewayClassName: nginx
  listeners:
  - name: codeify
    port: 80
    protocol: HTTP
    hostname: "*.codeify.me"
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: "true"
  - name: codeify-https
    port: 443
    protocol: HTTPS
    hostname: "*.codeify.me"
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: "true"
    tls:
      mode: Terminate
      certificateRefs:
        - name: codeify-me-tls
          kind: Secret
```

## 6. Example Routes (`cross-namespace-route.yaml`)
Two hostnames:
- `application.codeify.me` → Service `nginx-server` in `application`
- `route-app.codeify.me` → Service `route-app` in `route-app`

Path match strategy:
```yaml
rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
```
This covers all paths under the root.

## 7. Apply Sequence
**source code :- nginx-fabric/deployment-service-file**

```bash
# 1. Create namespaces with label
kubectl apply nginx-fabric/deployment-service-file/application-ns.yaml
kubectl apply nginx-fabric/deployment-service-file/route-ns.yaml

# 2. Deployment and  services k8s
kubectl apply -f nginx-fabric/deployment-service-file/deployment-service.yaml
kubectl apply -f nginx-fabric/deployment-service-file/deploy+service-2.yaml

# 3. Deploy Gateway (if not already)
kubectl apply -f nginx-fabric/nginx-gateway.yaml

# 4. Create cross-namespace routes
kubectl apply -f nginx-fabric/cross-namespace/cross-namespace-route.yaml

# 5. Verify
kubectl get httproute -A
kubectl describe httproute application-route -n application
kubectl describe httproute route-app -n route-app
```


## 8. Troubleshooting
| Issue | Check | Action |
|-------|-------|--------|
| Route not Accepted | `kubectl describe httproute` | Confirm namespace label or loosen policy to `from: All` |
| Wrong listener bound | `sectionName` mismatch | Use correct listener name from Gateway spec |
| Backend 404/503 | Service name/port | Verify service exists & selector matches pod labels |
| TLS errors | Certificate secret | Ensure cert-manager issued secret referenced by Gateway |

Command for detailed route diagnostics:
```bash
kubectl describe httproute <route-name> -n <namespace>
```
Look at `Parent Refs` and `Conditions` sections for acceptance reasons.

## 9. Security Considerations
- Use selector-based policy instead of `from: All` to restrict which namespaces can attach routes.
- Limit wildcard hostnames if subdomain issuance should be controlled.
- Separate internal vs external apps by hostname patterns (e.g. `internal.*`).

## 10. Extending Pattern
- Add additional namespaces by labeling them `gateway-access: "true"`.
- Introduce `TCPRoute` or `GRPCRoute` for other protocols.


## 11. Reference Map
```
Gateway:        nginx-fabric/nginx-gateway.yaml
Namespaces:     nginx-fabric/deployment-service-file/*-ns.yaml
Services:       nginx-fabric/deployment-service-file/*.yaml
Routes:         nginx-fabric/cross-namespace/cross-namespace-route.yaml
Certs:          nginx-fabric/cert-manager-letsencrypt/certmanager-nginxgateway-issuer.yaml
```

---
