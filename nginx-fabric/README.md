# Codeify Hub

A collection of Kubernetes configurations, tutorials, and best practices for cloud-native deployments.

## 📁 nginx-fabric

Production-ready NGINX Gateway API setup with Let's Encrypt TLS automation and cross-namespace routing.

### Highlights

- **🚪 Gateway API Implementation**: NGINX Gateway controller with HTTP/HTTPS listeners for `*.codeify.me`
- **🔒 Automated TLS Certificates**: Let's Encrypt integration via cert-manager with DNS-01 (wildcard) and HTTP-01 solvers
- **🌐 Cross-Namespace Routing**: Multi-tenant HTTPRoute configuration with label-based access control
- **☁️ Cloud LoadBalancer**: Automatic external IP provisioning for production ingress

### Key Components

| Component | Description | Path |
|-----------|-------------|------|
| **Gateway** | Central ingress with TLS termination | `nginx-fabric/cross-namespace/nginx-gateway.yaml` |
| **Cert Manager** | Let's Encrypt issuer configuration | `nginx-fabric/cert-manager-letsencrypt/` |
| **Cross-Namespace Routes** | Multi-tenant HTTPRoute examples | `nginx-fabric/cross-namespace/` |
| **Sample Apps** | Demo deployments & services | `nginx-fabric/deployment-service-file/` |

### Quick Start

```bash
# 1. Install cert-manager with Gateway API support
helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager --create-namespace \
  --set config.enableGatewayAPI=true

# 2. Create namespaces with access labels
kubectl apply -f deployment-service-file/application-ns.yaml
kubectl apply -f deployment-service-file/route-ns.yaml

# 3. Deploy cert-manager issuer
kubectl apply -f cert-manager-letsencrypt/certmanager-nginxgateway-issuer.yaml

# 4. Deploy gateway
kubectl apply -f cross-namespace/nginx-gateway.yaml

# 5. Deploy sample applications
kubectl apply -f deployment-service-file/deployment-service.yaml
kubectl apply -f deployment-service-file/deploy+service-2.yaml

# 6. Create routes
kubectl apply -f cross-namespace/cross-namespace-route.yaml

# 7. Verify
kubectl get gateway -n nginx-gateway
kubectl get certificate -n nginx-gateway
kubectl get httproute -A
```

### Documentation

- **[📖 Complete Focus Guide](docs/FOCUS-DOCS.md)** - Comprehensive architecture, prerequisites, and operational reference
- **[🔐 Cert-Manager Setup](nginx-fabric/cert-manager-letsencrypt/README.md)** - TLS certificate automation with DNS-01/HTTP-01 solvers
- **[🔀 Cross-Namespace Routing](nginx-fabric/cross-namespace/README.md)** - Multi-tenant ingress patterns and troubleshooting

### Architecture Overview

```
┌─────────────────────────────────────────────┐
│         Cloud LoadBalancer (External IP)     │
└────────────────┬────────────────────────────┘
                 │
         ┌───────▼────────┐
         │  nginx-gateway │  (Gateway API)
         │  Namespace     │  - HTTP :80
         │                │  - HTTPS :443 (TLS Terminate)
         └───────┬────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼────┐  ┌───▼────┐  ┌───▼────┐
│ App NS │  │Route NS│  │ ...    │
│(labeled)│  │(labeled)│  │        │
└────────┘  └────────┘  └────────┘
```

### Features

✅ **Wildcard TLS**: `*.codeify.me` via DNS-01 validation  
✅ **Namespace Isolation**: Selector-based route attachment (`gateway-access: "true"`)  
✅ **Multiple Backends**: Path-based routing to services across namespaces  
✅ **Auto-Renewal**: cert-manager handles certificate lifecycle  
✅ **Production Ready**: Configured for DigitalOcean DNS (adaptable to other providers)

### Prerequisites

- Kubernetes cluster (1.28+)
- Gateway API CRDs installed
- NGINX Gateway controller
- cert-manager (with Gateway API enabled)
- DNS provider API token (for wildcard certs)

### Use Cases

- Multi-tenant SaaS platforms with namespace-per-customer isolation
- Microservices ingress with centralized TLS management
- Development/staging environments sharing single LoadBalancer
- Tutorial/blog demonstrations of Gateway API patterns

---

**License**: See [LICENSE](LICENSE)  
**Repository**: [github.com/sanathwaghela/codeify-hub](https://github.com/sanathwaghela/codeify-hub)
