# Codeify Hub

A comprehensive collection of production-ready Kubernetes configurations, cloud-native tools, and infrastructure-as-code templates for modern DevOps workflows.

## 📂 Repository Structure

This repository is organized by technology stack and use case. Each folder contains complete setup guides, manifests, and best practices.

### 🌐 nginx-fabric/
**NGINX Gateway API + cert-manager + Cross-Namespace Routing**

Production-ready Gateway API implementation with automated TLS certificates and multi-tenant ingress patterns.

- Gateway API configuration with NGINX controller
- Let's Encrypt certificate automation (DNS-01 & HTTP-01)
- Multi-tenant HTTPRoute with namespace isolation
- Wildcard domain support (`*.example.com`)
- Sample microservices deployments

📖 **Documentation**: [nginx-fabric/README.md](nginx-fabric/README.md)

---

## 🤝 Contributing

This repository welcomes contributions! To add a new tool or configuration:

1. Create a new top-level folder (e.g., `istio-setup/`, `argocd-gitops/`)
2. Include a comprehensive `README.md` with setup instructions
3. Add manifests and configuration files
4. Update this root README with your addition
5. Submit a pull request

## 📝 Repository Standards

Each tool folder should include:
- Clear prerequisites section
- Step-by-step deployment guide
- Troubleshooting section
- Production considerations
- Example use cases

## 🔗 Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Gateway API Docs](https://gateway-api.sigs.k8s.io/)
- [cert-manager Docs](https://cert-manager.io/docs/)

---

**License**: [MIT](LICENSE)  
**Maintainer**: [@sanathwaghela](https://github.com/sanathwaghela)  
**Repository**: [github.com/sanathwaghela/codeify-hub](https://github.com/sanathwaghela/codeify-hub)
