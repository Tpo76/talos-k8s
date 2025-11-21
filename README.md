# Talos Kubernetes Cluster - GitOps Repository

This repository contains the GitOps configuration for a Talos Linux Kubernetes cluster, managed by FluxCD.

## 🏗️ Architecture

- **Kubernetes Distribution**: Talos Linux
- **Cluster Management**: [Omni](https://omni.int.centreon.com:8100) (OIDC/SAML authentication)
- **GitOps**: FluxCD v2.7.2 (continuous reconciliation from Git)
- **Secret Management**: SOPS with Age encryption
- **Load Balancing**: MetalLB v0.17.2 (bare-metal LoadBalancer support)
- **Remote Access**: Tailscale subnet router

## 📁 Repository Structure

```
talos-k8s/
├── clusters/staging/          # Cluster-specific Flux configurations
├── infrastructure/
│   ├── base/                  # Shared infrastructure (MetalLB, Tailscale)
│   └── staging/               # Staging environment overlays
├── apps/
│   ├── base/                  # Shared application definitions
│   └── staging/               # Staging environment overlays
└── docs/
    ├── CLAUDE.md              # Technical reference for Claude Code
    ├── SOPS-SETUP.md          # SOPS encryption setup guide
    └── QUICKSTART-SOPS.md     # SOPS quick reference
```

## 🚀 Quick Start

### Prerequisites

1. **Cluster Access**: OIDC authentication via Omni
2. **Tools**:
   - `kubectl` - Kubernetes CLI
   - `kubectl oidc-login` - OIDC authentication plugin
   - `flux` - FluxCD CLI
   - `sops` - Secret encryption (for managing secrets)
   - `age` - Encryption tool for SOPS

### Initial Setup

1. **Configure kubectl** to connect via Omni:
   ```bash
   # Kubeconfig is pre-configured to use Omni at omni.int.centreon.com:8100
   kubectl get nodes
   ```

2. **Verify Flux is running**:
   ```bash
   flux check
   flux get all
   ```

3. **Set up SOPS** (for managing encrypted secrets):
   ```bash
   # See QUICKSTART-SOPS.md for detailed instructions
   age-keygen -o ~/.config/sops/age/keys.txt
   ```

## 🔄 GitOps Workflow

All changes to the cluster are made through Git:

1. **Make changes** to manifests in this repository
2. **Commit and push** to the `main` branch
3. **Flux automatically reconciles** changes to the cluster (within 10 minutes)
4. **Monitor** deployment: `flux get kustomizations`

### Example: Adding a New Application

```bash
# 1. Create application manifests
mkdir -p apps/base/myapp
cat > apps/base/myapp/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
EOF

# 2. Create kustomization
cat > apps/base/myapp/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
EOF

# 3. Add to apps/base/kustomization.yaml
echo "  - myapp" >> apps/base/kustomization.yaml

# 4. Validate
kubectl kustomize apps/staging

# 5. Commit and push
git add apps/base/myapp apps/base/kustomization.yaml
git commit -m "feat: add myapp deployment"
git push

# 6. Monitor deployment
flux reconcile kustomization apps
kubectl get pods -n default -w
```

## 🔐 Secret Management

Secrets are encrypted with SOPS before committing to Git:

```bash
# Create a secret
cat > secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
stringData:
  password: "super-secret-value"
EOF

# Encrypt with SOPS
sops --encrypt --in-place secret.yaml

# Now safe to commit
git add secret.yaml
git commit -m "Add encrypted secret"
git push
```

**See [SOPS-SETUP.md](SOPS-SETUP.md) for complete setup instructions.**

## 🌐 Remote Access

### From Office
- Direct access via office network
- kubectl connects through Omni automatically

### Remotly
1. **Install Tailscale** on your machine
2. **Connect** to your Tailscale network
3. **Add to hosts file**:
   ```bash
   # Windows (as Administrator)
   Add-Content C:\Windows\System32\drivers\etc\hosts "<OMNI_IP> omni.int.centreon.com"

   # Linux/Mac
   echo "<OMNI_IP> omni.int.centreon.com" | sudo tee -a /etc/hosts
   ```
4. **Use kubectl normally** - traffic routes through Tailscale subnet router

## 📊 Monitoring & Troubleshooting

### Check Cluster Status
```bash
# Flux status
flux get all

# All resources
kubectl get all -A

# Specific components
kubectl get all -n metallb-system
kubectl get all -n tailscale
```

### Common Issues

**Flux reconciliation stuck:**
```bash
flux logs --level=error
kubectl describe kustomization -n flux-system
```

**Secret decryption failing:**
```bash
# Verify SOPS age secret exists
kubectl get secret sops-age -n flux-system

# Check decryption logs
kubectl logs -n flux-system deploy/kustomize-controller | grep -i sops
```

**LoadBalancer service pending:**
```bash
# Check MetalLB status
kubectl get pods -n metallb-system
kubectl get ipaddresspool,l2advertisement -n metallb-system
kubectl logs -n metallb-system -l component=controller
```

## 📚 Documentation

- **[SOPS-SETUP.md](SOPS-SETUP.md)** - Complete SOPS encryption setup guide
- **[QUICKSTART-SOPS.md](QUICKSTART-SOPS.md)** - SOPS quick reference

## 🔧 Infrastructure Components

### MetalLB
- **Version**: v0.17.2
- **Configuration**: CRD-based (IPAddressPool + L2Advertisement)
- **IP Range**: 192.168.0.200-250
- **Purpose**: Provides LoadBalancer service type for bare-metal

### Tailscale
- **Purpose**: Remote cluster access via VPN
- **Routes**: Office LAN + K8s services
- **Authentication**: SOPS-encrypted auth key

### Flux
- **Version**: v2.7.2
- **Reconciliation**: Every 10 minutes (infrastructure), 1 minute (apps)
- **Features**: SOPS decryption, automatic secret management

## 🤝 Contributing

1. **Create a feature branch** (optional for small changes)
2. **Make your changes** in the appropriate directory:
   - Infrastructure: `infrastructure/base/` or `infrastructure/staging/`
   - Applications: `apps/base/` or `apps/staging/`
3. **Validate** locally: `kubectl kustomize <directory>`
4. **Encrypt secrets** if needed: `sops --encrypt --in-place secret.yaml`
5. **Commit** following conventional commits format:
   - `feat:` - New feature
   - `fix:` - Bug fix
   - `refactor:` - Code restructure
   - `docs:` - Documentation
6. **Push** and monitor Flux reconciliation

## 🆘 Support

- **Cluster Issues**: Check Flux logs and kustomization status
- **Authentication Issues**: Verify Omni access and OIDC configuration
- **Network Issues**: Check Tailscale connection and routes
- **Secret Issues**: Verify SOPS age secret exists in cluster

## 🔗 Useful Links

- **Omni Console**: https://omni.int.centreon.com:8100/
- **GitHub Repository**: https://github.com/Tpo76/talos-k8s
- **Tailscale Admin**: https://login.tailscale.com/admin/machines
- **Flux Documentation**: https://fluxcd.io/
- **SOPS Documentation**: https://github.com/mozilla/sops

## 📝 License

This is an internal infrastructure repository.
