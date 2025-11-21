# Tailscale Subnet Router for Kubernetes

This deployment sets up a Tailscale subnet router inside your Kubernetes cluster, allowing you to access cluster resources (services, API server) remotely through your Tailscale network.

## Prerequisites

1. A Tailscale account (free tier works fine)
2. Access to create auth keys in your Tailscale admin console

## Setup Instructions

### Step 1: Generate Tailscale Auth Key

1. Go to https://login.tailscale.com/admin/settings/keys
2. Click "Generate auth key"
3. Configure the key:
   - **Reusable**: ✅ Enabled (allows the pod to restart)
   - **Ephemeral**: ✅ Enabled (device disappears when pod stops)
   - **Pre-approved**: ✅ Enabled (optional, but recommended)
   - **Expiration**: Set to your preference (90 days recommended)
4. Copy the generated key (starts with `tskey-auth-`)

### Step 2: Create the Secret

**IMPORTANT**: Do this BEFORE committing/pushing the deployment to avoid errors.

You have two options:

#### Option A: Using kubectl (requires current cluster access)

```bash
kubectl create namespace tailscale
kubectl create secret generic tailscale-auth \
  --from-literal=TS_AUTH_KEY=tskey-auth-YOUR_KEY_HERE \
  -n tailscale
```

#### Option B: Using a secret file (for GitOps)

1. Copy the template:
   ```bash
   cp tailscale-auth-secret.yaml.template tailscale-auth-secret.yaml
   ```

2. Edit `tailscale-auth-secret.yaml` and replace `YOUR_TAILSCALE_AUTH_KEY_HERE` with your actual key

3. **DO NOT commit this file!** Add it to `.gitignore` or use a secret management solution like:
   - Sealed Secrets
   - SOPS with age/GPG
   - External Secrets Operator
   - Vault

4. Apply manually:
   ```bash
   kubectl apply -f tailscale-auth-secret.yaml
   ```

### Step 3: Configure Routes

The subnet router advertises:
- `10.96.0.0/12` - Kubernetes service network
- `192.168.0.0/24` - Office LAN (required to reach Omni at 192.168.0.102)

**IMPORTANT:** The office LAN route is **critical** because Omni (https://omni.int.centreon.com → 192.168.0.102:8100) is only accessible from the office network. Without this route, you cannot use kubectl from home.

**To verify/update your cluster's actual networks:**

```bash
# Service network (ClusterIP range)
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range

# Pod network
kubectl cluster-info dump | grep -m 1 cluster-cidr

# Office LAN network (check node IPs)
kubectl get nodes -o wide
```

**Update the routes in `tailscale-subnet-router.yaml` if needed:**

Edit the `TS_ROUTES` environment variable:

```yaml
- name: TS_ROUTES
  value: "10.96.0.0/12,192.168.0.0/24,10.244.0.0/16"  # service, office LAN, pod network
```

### Step 4: Deploy via GitOps

Once the secret is created, commit and push:

```bash
git add infrastructure/base/tailscale/
git commit -m "feat: add Tailscale subnet router for remote cluster access"
git push
```

Flux will automatically deploy Tailscale within 10 minutes, or force reconciliation:

```bash
flux reconcile kustomization infrastructure
```

### Step 5: Enable Subnet Routes in Tailscale Admin

1. Go to https://login.tailscale.com/admin/machines
2. Find the new device named `k8s-subnet-router`
3. Click the **⋯** menu → **Edit route settings**
4. Enable the advertised subnet routes (approve them)

### Step 6: Access Your Cluster from Home

Once routes are approved, you can access your cluster from home!

#### Option A: Use Your Existing Kubeconfig (Recommended)

Your existing kubeconfig connects through Omni at `https://omni.int.centreon.com:8100` (192.168.0.102). Since Tailscale now routes the office LAN, this will work from home!

**Add Omni to your home machine's hosts file:**

Windows (run as Admin):
```powershell
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "192.168.0.102 omni.int.centreon.com"
```

Linux/Mac:
```bash
echo "192.168.0.102 omni.int.centreon.com" | sudo tee -a /etc/hosts
```

**Then use kubectl normally:**
```bash
kubectl get nodes
kubectl get pods -A
```

Your existing OIDC authentication through Omni will work exactly as it does at the office!

#### Option B: Direct API Access (Alternative)

You can also access the Kubernetes API directly via its ClusterIP (bypassing Omni):

```bash
# Find the API server ClusterIP
kubectl get svc kubernetes -n default

# Access directly (usually 10.96.0.1)
kubectl --server=https://10.96.0.1:443 get nodes --insecure-skip-tls-verify
```

Note: This requires valid cluster certificates or a token for authentication.

## Troubleshooting

### Check Tailscale Pod Status

```bash
kubectl get pods -n tailscale
kubectl logs -n tailscale deployment/tailscale-subnet-router
```

### Verify Routes Are Advertised

```bash
kubectl logs -n tailscale deployment/tailscale-subnet-router | grep -i routes
```

### Test Connectivity

From a device on your Tailscale network:

```bash
# Ping the Kubernetes API service
ping 10.96.0.1

# Test HTTPS connection
curl -k https://10.96.0.1:443
```

## Security Considerations

1. **Auth Key**: Never commit the auth key to Git
2. **Routes**: Only advertise networks you need to access
3. **Tailscale ACLs**: Use Tailscale ACLs to restrict access to specific users/devices
4. **Pod Security**: The pod requires NET_ADMIN capability for IP forwarding
5. **API Access**: Consider using RBAC to limit what can be done via the API

## Accessing Specific Services

### Example: Access nginx LoadBalancer service

```bash
# Get the service IP
kubectl get svc nginx-service

# Access from remote machine
curl http://<service-ip>
```

## Alternative: API Server Proxy

If you only need kubectl access, consider using `kubectl proxy` or port-forwarding instead of a full subnet router.
