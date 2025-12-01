# Longhorn Installation on Talos via Omni

## Prerequisites

Longhorn on Talos requires:
1. **iSCSI system extension** - Required for Longhorn's storage backend
2. **Dedicated storage disk** (recommended) - Can be additional disk or partition
3. **Machine configuration updates** - Applied via Omni

## Step 1: Configure Talos Machine Config via Omni

### 1.1 Add iSCSI System Extension

In Omni (https://omni.int.centreon.com:8100):

1. Go to your cluster → **Machine Configuration**
2. Click **Edit Config**
3. Add the `iscsi-tools` system extension:

```yaml
machine:
  install:
    extensions:
      - image: ghcr.io/siderolabs/iscsi-tools:v0.1.5
```

### 1.2 Configure Storage Disk for Longhorn

**Option A: Use Existing Disk with Directory**
If you want to use existing system disk (simpler, but shared storage):

```yaml
machine:
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - bind
          - rshared
          - rw
```

**Option B: Dedicated Disk (Recommended)**
If you're adding a new disk (e.g., `/dev/sdb`) for Longhorn:

```yaml
apiVersion: v1alpha1
kind: UserVolumeConfig
name: longhorn
provisioning:
  diskSelector:
    match: disk.dev_path == "/dev/sdb"
  minSize: 10GB  # Required: minimum size
filesystem:
  type: xfs

---
# Document 2: Machine config for kubelet mounts

machine:
  kubelet:
    extraMounts:
      - destination: /var/mnt/longhorn
        type: bind
        source: /var/mnt/longhorn
        options:
          - bind
          - rshared
          - rw
```

**Important Notes:**
- Check your disk device name with: `talosctl -n <node-ip> get disks` (via Omni console)
- The `rshared` mount option is **critical** for Longhorn to work
- All worker nodes should have the same configuration

### 1.3 Apply Configuration via Omni

1. Save the machine config in Omni
2. Omni will rolling-update the nodes (they will reboot)
3. Wait for all nodes to come back online
4. Verify the extension is loaded:
   - In Omni console: Check node status shows extensions loaded
   - Or via talosctl: `talosctl -n <node-ip> get extensions`

### 1.4 Verify iSCSI is Running

After nodes reboot, check iSCSI service is running:

```bash
# Via Omni console or talosctl
talosctl -n <node-ip> service iscsid status
```

Should show: `STATE: Running`

## Step 2: Deploy Longhorn via Flux

Once Talos configuration is applied and nodes are ready, deploy Longhorn using the Flux HelmChart configuration created in this repository.

See the `infrastructure/base/longhorn/` directory for:
- `namespace.yaml` - Longhorn namespace
- `helmrelease.yaml` - Flux HelmRelease for Longhorn
- `kustomization.yaml` - Kustomize configuration

## Step 3: Verify Longhorn Installation

```bash
# Check Longhorn pods (should all be Running)
kubectl get pods -n longhorn-system

# Check Longhorn nodes (should show all worker nodes as "Schedulable")
kubectl get nodes.longhorn.io -n longhorn-system

# Access Longhorn UI (via port-forward)
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
# Then open: http://localhost:8080

# Create test PVC to verify storage works
kubectl apply -f infrastructure/base/longhorn/test-pvc.yaml
kubectl get pvc -n default
```

## Step 4: Set Longhorn as Default StorageClass (Optional)

```bash
# Set Longhorn as default
kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
# Should show "longhorn (default)"
```

## Troubleshooting

### Pods stuck in ContainerCreating
- **Check iSCSI**: `talosctl -n <node-ip> service iscsid status`
- **Check extensions**: `talosctl -n <node-ip> get extensions`
- **Check kubelet logs**: `talosctl -n <node-ip> logs kubelet`

### Nodes show "Unschedulable" in Longhorn UI
- Verify `/var/lib/longhorn` mount exists: `talosctl -n <node-ip> ls /var/lib/longhorn`
- Check mount options include `rshared`: `talosctl -n <node-ip> mounts | grep longhorn`

### "No default storage class" warnings
- This is normal initially, Longhorn creates the StorageClass after installation
- Wait a few minutes, then check: `kubectl get storageclass`

## Additional Resources

- [Talos System Extensions](https://www.talos.dev/latest/talos-guides/configuration/system-extensions/)
- [Longhorn Talos Support](https://longhorn.io/docs/latest/advanced-resources/os-distro-specific/talos-linux-support/)
- [Longhorn Installation](https://longhorn.io/docs/latest/deploy/install/)
