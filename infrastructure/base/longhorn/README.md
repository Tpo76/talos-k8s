# Longhorn - Cloud-Native Distributed Block Storage

Longhorn is a lightweight, reliable, and highly available distributed block storage system for Kubernetes.

## Overview

This directory contains the Flux HelmRelease configuration for deploying Longhorn on Talos Linux.

**Version:** 1.7.2 (Latest stable as of January 2025)

## Components

- `namespace.yaml` - Creates `longhorn-system` namespace
- `helmrepository.yaml` - Flux HelmRepository pointing to https://charts.longhorn.io
- `helmrelease.yaml` - Flux HelmRelease with Talos-optimized configuration
- `test-pvc.yaml` - Test PVC and Pod to verify storage works (not deployed by default)
- `kustomization.yaml` - Kustomize configuration

## Prerequisites (CRITICAL!)

**Before deploying Longhorn, you MUST configure Talos machine config via Omni:**

1. **iSCSI system extension** - Required for Longhorn to work
2. **Storage disk configuration** - Dedicated mount at `/var/lib/longhorn`

See `LONGHORN-TALOS-SETUP.md` in the repository root for detailed instructions.

## Configuration Details

### Default Settings
- **Default StorageClass:** `longhorn` (set as cluster default)
- **Replica Count:** 3 (for high availability)
- **Data Path:** `/var/lib/longhorn` (Talos standard location)
- **Reclaim Policy:** Retain (data persists after PVC deletion)

### UI Access
- **Service Type:** ClusterIP (default)
- **Access Method:**
  ```bash
  kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
  ```
  Then open: http://localhost:8080

### Exposing UI via MetalLB
To expose Longhorn UI via LoadBalancer, edit `helmrelease.yaml`:

```yaml
service:
  ui:
    type: LoadBalancer  # Change from ClusterIP
```

### Ingress Configuration
To expose via Ingress (requires ingress controller), edit `helmrelease.yaml`:

```yaml
ingress:
  enabled: true
  host: longhorn.yourdomain.com
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

## Deployment

Once Talos machine config is applied (see `LONGHORN-TALOS-SETUP.md`):

```bash
# Validate configuration
kubectl kustomize infrastructure/staging

# Commit and push (Flux will reconcile automatically)
git add infrastructure/base/longhorn/
git commit -m "feat: add Longhorn distributed storage"
git push origin main

# Or force immediate reconciliation
flux reconcile source git flux-system
flux reconcile kustomization infrastructure
```

## Verification

```bash
# Check Longhorn pods (all should be Running)
kubectl get pods -n longhorn-system

# Check Longhorn nodes (should show all worker nodes)
kubectl get nodes.longhorn.io -n longhorn-system

# Check StorageClass
kubectl get storageclass
# Should show: longhorn (default)

# Test storage with test PVC
kubectl apply -f infrastructure/base/longhorn/test-pvc.yaml
kubectl get pvc -n default
kubectl logs -n default longhorn-test-pod
```

## Resource Usage

**Per Node:**
- **Longhorn Manager:** ~100-500 MB RAM, minimal CPU
- **Longhorn Engine:** ~10-50 MB per volume
- **CSI Driver:** ~50 MB RAM, minimal CPU

**Cluster-wide:**
- **UI:** ~50 MB RAM
- **Helm chart components:** ~200 MB total

## Troubleshooting

### Pods stuck in ContainerCreating
```bash
# Check iSCSI service on Talos node
talosctl -n <node-ip> service iscsid status

# Check kubelet logs
talosctl -n <node-ip> logs kubelet
```

### Nodes show "Unschedulable"
```bash
# Verify mount exists
talosctl -n <node-ip> ls /var/lib/longhorn

# Check mount options (must include 'rshared')
talosctl -n <node-ip> mounts | grep longhorn
```

### Volume creation fails
```bash
# Check Longhorn manager logs
kubectl logs -n longhorn-system -l app=longhorn-manager

# Check instance manager logs
kubectl logs -n longhorn-system -l app=longhorn-instance-manager
```

## Upgrading Longhorn

To upgrade Longhorn version:

1. Check [Longhorn releases](https://github.com/longhorn/longhorn/releases)
2. Update `version` in `helmrelease.yaml`:
   ```yaml
   chart:
     spec:
       version: 1.8.0  # New version
   ```
3. Commit and push (Flux will upgrade automatically)
4. Monitor: `flux get helmreleases -n longhorn-system --watch`

## Uninstalling

⚠️ **WARNING:** This will delete all Longhorn volumes and data!

```bash
# Delete all PVCs using Longhorn
kubectl get pvc -A | grep longhorn

# Suspend Flux reconciliation
flux suspend kustomization infrastructure

# Remove Longhorn from kustomization
# Edit infrastructure/base/kustomization.yaml, remove 'longhorn'

# Force cleanup
kubectl delete namespace longhorn-system --wait=false
kubectl patch namespace longhorn-system -p '{"metadata":{"finalizers":null}}'

# Resume Flux
flux resume kustomization infrastructure
```

## Additional Resources

- [Longhorn Documentation](https://longhorn.io/docs/)
- [Longhorn Talos Support](https://longhorn.io/docs/latest/advanced-resources/os-distro-specific/talos-linux-support/)
- [Longhorn Best Practices](https://longhorn.io/docs/latest/best-practices/)