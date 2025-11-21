# SOPS + Age Encryption Setup for Flux

This guide shows how to encrypt secrets with SOPS and Age so you can safely commit them to Git, and Flux will decrypt them automatically in the cluster.

## Step 1: Install Tools (Run in Admin Terminal)

```powershell
# Install Age encryption (portable version)
choco install age.portable -y

# Install SOPS
choco install sops -y

# Restart your terminal after installation
```

Verify installation:
```bash
age --version
sops --version
```

## Step 2: Generate Age Key Pair

```bash
# Create a directory for age keys
mkdir -p ~/.config/sops/age

# Generate a new age key pair
age-keygen -o ~/.config/sops/age/keys.txt

# View the generated key
cat ~/.config/sops/age/keys.txt
```

This will output something like:
```
# created: 2024-01-01T12:00:00Z
# public key: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AGE-SECRET-KEY-1XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXx
```

**IMPORTANT**:
- The line starting with `age1...` is your PUBLIC key (safe to share, used for encryption)
- The line starting with `AGE-SECRET-KEY-1...` is your PRIVATE key (KEEP SECRET!)

## Step 3: Store the Age Private Key in the Cluster

**THIS REQUIRES CLUSTER ACCESS** - You'll need to do this from the office or have someone else do it.

```bash
# Get the private key (the line starting with AGE-SECRET-KEY-1)
cat ~/.config/sops/age/keys.txt | grep "^AGE-SECRET-KEY-1"

# Create the secret in the cluster
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-literal=age.agekey='<YOUR_PRIVATE_KEY_HERE>'

# Verify the secret was created
kubectl get secret sops-age -n flux-system
```

## Step 4: Configure SOPS

Create `.sops.yaml` in the repository root:

```yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  # REPLACE with your public key
```

**REPLACE** `age1xxx...` with YOUR public key from Step 2!

## Step 5: Encrypt the Tailscale Secret

Create the secret file:

```bash
cat > infrastructure/base/tailscale/tailscale-auth-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-auth
  namespace: tailscale
type: Opaque
stringData:
  TS_AUTH_KEY: "tskey-auth-YOUR_TAILSCALE_KEY_HERE"
EOF
```

Encrypt it with SOPS:

```bash
sops --encrypt --in-place infrastructure/base/tailscale/tailscale-auth-secret.yaml
```

The file is now encrypted! You can safely commit it. View it:

```bash
cat infrastructure/base/tailscale/tailscale-auth-secret.yaml
```

You'll see encrypted content like:
```yaml
apiVersion: v1
kind: Secret
metadata:
    name: tailscale-auth
    namespace: tailscale
type: Opaque
stringData:
    TS_AUTH_KEY: ENC[AES256_GCM,data:xxx,iv:xxx,tag:xxx,type:str]
sops:
    kms: []
    ...
```

## Step 6: Update Kustomization to Include Secret

```bash
# Add the encrypted secret to kustomization
echo "  - tailscale-auth-secret.yaml" >> infrastructure/base/tailscale/kustomization.yaml
```

## Step 7: Configure Flux to Decrypt

Update the Flux Kustomization to enable decryption - see the updated file:
`clusters/stagging/infrastructure.yaml`

## Step 8: Commit and Push

```bash
git add .sops.yaml infrastructure/base/tailscale/
git commit -m "feat: add encrypted Tailscale secret with SOPS"
git push
```

## Daily Usage

### Encrypting a New Secret

```bash
# Create the secret file
cat > my-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
stringData:
  password: "super-secret"
EOF

# Encrypt it
sops --encrypt --in-place my-secret.yaml

# Commit it
git add my-secret.yaml
git commit -m "Add encrypted secret"
git push
```

### Editing an Encrypted Secret

```bash
# SOPS will decrypt, open your editor, then re-encrypt on save
sops infrastructure/base/tailscale/tailscale-auth-secret.yaml
```

### Viewing an Encrypted Secret

```bash
# Decrypt and view (doesn't modify the file)
sops --decrypt infrastructure/base/tailscale/tailscale-auth-secret.yaml
```

## Troubleshooting

### "Failed to get the data key"
- Make sure `~/.config/sops/age/keys.txt` exists and contains your private key
- Check that the public key in `.sops.yaml` matches your private key

### Flux can't decrypt
- Verify the sops-age secret exists: `kubectl get secret sops-age -n flux-system`
- Check Flux logs: `kubectl logs -n flux-system deploy/kustomize-controller`
- Ensure the infrastructure Kustomization has `decryption` configured

### "No decryption key found"
- The cluster needs the private key in the `sops-age` secret
- You encrypted with a different public key than what's in the cluster

## Security Notes

1. **NEVER commit** `~/.config/sops/age/keys.txt` or the private key
2. **Backup** your age private key securely (password manager, encrypted backup)
3. **Rotate keys** periodically by generating a new key pair and re-encrypting secrets
4. The `.sops.yaml` file CAN be committed (it only contains the public key)
5. If you lose the private key, you'll need to regenerate all secrets

## References

- [SOPS Documentation](https://github.com/mozilla/sops)
- [Age Encryption](https://github.com/FiloSottile/age)
- [Flux SOPS Guide](https://fluxcd.io/flux/guides/mozilla-sops/)
