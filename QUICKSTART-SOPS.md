# Quick Start: Encrypt Tailscale Secret with SOPS

Follow these steps in order:

## 1. Install Tools (Admin Terminal)

```powershell
choco install age.portable sops -y
```

Close and reopen your terminal, then verify:

```bash
age --version
sops --version
```

## 2. Generate Age Key

```bash
# Create directory
mkdir -p ~/.config/sops/age

# Generate key
age-keygen -o ~/.config/sops/age/keys.txt

# View your keys
cat ~/.config/sops/age/keys.txt
```

**Copy both keys from the output:**
- Line starting with `age1...` = PUBLIC key (for .sops.yaml)
- Line starting with `AGE-SECRET-KEY-1...` = PRIVATE key (for cluster)

## 3. Create .sops.yaml

```bash
# Copy template
cp .sops.yaml.template .sops.yaml

# Edit .sops.yaml and replace YOUR_AGE_PUBLIC_KEY_HERE with your public key (age1...)
```

Or use sed (replace the key):
```bash
sed 's/YOUR_AGE_PUBLIC_KEY_HERE/age1your_actual_public_key_here/' .sops.yaml.template > .sops.yaml
```

## 4. Store Private Key in Cluster

**IMPORTANT**: This requires cluster access. Do this from the office or have someone run it:

```bash
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-literal=age.agekey='AGE-SECRET-KEY-1XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX'
```

Replace `AGE-SECRET-KEY-1XXX...` with your actual private key!

## 5. Get Tailscale Auth Key

1. Go to: https://login.tailscale.com/admin/settings/keys
2. Click "Generate auth key"
3. Enable: Reusable, Ephemeral, Pre-approved
4. Copy the key (starts with `tskey-auth-`)

## 6. Create and Encrypt Tailscale Secret

```bash
# Create the secret file
cat > infrastructure/base/tailscale/tailscale-auth-secret.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-auth
  namespace: tailscale
type: Opaque
stringData:
  TS_AUTH_KEY: "tskey-auth-PASTE_YOUR_ACTUAL_KEY_HERE"
EOF

# Encrypt it with SOPS
sops --encrypt --in-place infrastructure/base/tailscale/tailscale-auth-secret.yaml

# Verify it's encrypted (you should see ENC[...] in the output)
cat infrastructure/base/tailscale/tailscale-auth-secret.yaml
```

## 7. Add Secret to Kustomization

```bash
# Add the secret to the tailscale kustomization
cat >> infrastructure/base/tailscale/kustomization.yaml <<'EOF'
  - tailscale-auth-secret.yaml
EOF
```

## 8. Commit and Push

```bash
# Stage files
git add .sops.yaml .gitignore
git add clusters/stagging/infrastructure.yaml clusters/stagging/apps.yaml
git add infrastructure/base/tailscale/

# Check what will be committed
git status

# Commit
git commit -m "feat: add SOPS encryption and encrypted Tailscale secret"

# Push
git push
```

## 9. Approve Tailscale Routes (After Deployment)

1. Wait ~10 minutes for Flux to deploy (or force: `flux reconcile kustomization infrastructure`)
2. Go to: https://login.tailscale.com/admin/machines
3. Find `k8s-subnet-router`
4. Click ⋯ → Edit route settings
5. Approve the subnet routes

## 10. Access Cluster Remotely

From any device with Tailscale:

```bash
# Access Kubernetes API
kubectl --server=https://10.96.0.1:443 get nodes
```

---

## Troubleshooting

### "no key found to decrypt"
- Make sure `.sops.yaml` has your correct public key
- Verify age key exists: `cat ~/.config/sops/age/keys.txt`

### Flux can't decrypt
- Check secret exists: `kubectl get secret sops-age -n flux-system`
- View Flux logs: `kubectl logs -n flux-system deploy/kustomize-controller -f`

### Want to edit the encrypted secret later?
```bash
sops infrastructure/base/tailscale/tailscale-auth-secret.yaml
```

### Want to view without editing?
```bash
sops --decrypt infrastructure/base/tailscale/tailscale-auth-secret.yaml
```
