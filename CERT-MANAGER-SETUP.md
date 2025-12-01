# cert-manager Setup Guide

This document describes the cert-manager installation and TLS certificate management for the Talos Kubernetes cluster.

## Overview

cert-manager is installed to provide automated certificate lifecycle management. Currently configured with a manually-managed wildcard certificate for `*.int.centreon.com`.

### Architecture

- **Version**: v1.16.2 (latest stable)
- **Deployment**: 2 replicas each for controller, webhook, and CA injector (HA setup)
- **Certificate**: Wildcard certificate for `*.int.centreon.com`
- **Secret Management**: SOPS-encrypted certificate stored in Git
- **Renewal**: Manual (once per year)

## Configuration

### Components

cert-manager consists of three main components:

1. **Controller** - Manages Certificate resources and issuers
2. **Webhook** - Validates cert-manager custom resources
3. **CA Injector** - Injects CA data into webhooks and API services

All components run with 2 replicas for high availability.

### Wildcard Certificate

The wildcard certificate for `*.int.centreon.com` is stored as a Kubernetes Secret in the `traefik` namespace and configured as Traefik's default TLS certificate.

**Secret name**: `wildcard-int-centreon-com-tls`
**Namespace**: `traefik`
**Covers**: `*.int.centreon.com` and `int.centreon.com`

## Initial Setup

### 1. Install cert-manager

cert-manager is installed via FluxCD:

```bash
# Reconcile infrastructure to install cert-manager
flux reconcile kustomization infrastructure --with-source

# Verify installation
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager
```

Expected output:
```
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-xxxxxxxxx-xxxxx               1/1     Running   0          1m
cert-manager-cainjector-xxxxxxxxx-xxxxx    1/1     Running   0          1m
cert-manager-webhook-xxxxxxxxx-xxxxx       1/1     Running   0          1m
```

### 2. Configure Wildcard Certificate

**Step 1: Prepare your certificate files**

You need:
- `certificate.crt` - Your wildcard certificate for `*.int.centreon.com`
- `private.key` - The corresponding private key

**Step 2: Update the secret template**

Edit `infrastructure/base/traefik/wildcard-cert-secret.yaml` and replace the placeholders with your actual certificate and key:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: wildcard-int-centreon-com-tls
  namespace: traefik
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    MIIFXzCCBEegAwIBAgIQCCVwp7... (your certificate here)
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    MIIEvQIBADANBgkqhkiG9w0BAQ... (your private key here)
    -----END PRIVATE KEY-----
```

**Step 3: Encrypt the secret with SOPS**

```bash
cd talos-k8s/infrastructure/base/traefik

# Encrypt the secret
sops --encrypt --in-place wildcard-cert-secret.yaml

# Verify encryption
cat wildcard-cert-secret.yaml  # Should show encrypted content
```

**Step 4: Commit and deploy**

```bash
git add wildcard-cert-secret.yaml
git commit -m "feat: add wildcard TLS certificate for *.int.centreon.com"
git push

# Reconcile
flux reconcile kustomization infrastructure --with-source
```

**Step 5: Verify the certificate is loaded**

```bash
# Check the secret exists
kubectl get secret wildcard-int-centreon-com-tls -n traefik

# Check Traefik is using it
kubectl logs -n traefik -l app.kubernetes.io/name=traefik | grep -i tls
```

## Usage

### Default Certificate Behavior

With the wildcard certificate configured as Traefik's default:

- **All HTTPS ingresses** automatically use the wildcard certificate
- **No need to specify** `tls.secretName` in Ingress resources
- **Works for any subdomain** of `int.centreon.com`

### Example: Basic HTTPS Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: default
spec:
  tls:
    - hosts:
        - myapp.int.centreon.com
      # secretName omitted - uses default wildcard cert
  rules:
    - host: myapp.int.centreon.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
```

### Example: IngressRoute with TLS

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: my-app
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`myapp.int.centreon.com`)
      kind: Rule
      services:
        - name: my-app-service
          port: 80
  tls: {}  # Uses default wildcard certificate
```

### HTTP to HTTPS Redirect

To automatically redirect all HTTP traffic to HTTPS, uncomment the redirect configuration in Traefik:

Edit `infrastructure/base/traefik/helmrelease.yaml`:

```yaml
ports:
  web:
    port: 8000
    expose:
      default: true
    exposedPort: 80
    protocol: TCP
    redirectTo:
      port: websecure  # Uncomment this section
```

Then commit and reconcile:

```bash
git add infrastructure/base/traefik/helmrelease.yaml
git commit -m "feat: enable HTTP to HTTPS redirect"
git push
flux reconcile kustomization infrastructure --with-source
```

## Certificate Renewal (Once Per Year)

When your wildcard certificate expires (once per year), follow these steps to renew it:

### Step 1: Obtain New Certificate

Obtain a new wildcard certificate for `*.int.centreon.com` from your certificate authority.

### Step 2: Update the Secret

**Option A: Using the Template (Recommended)**

```bash
cd talos-k8s/infrastructure/base/traefik

# Copy the template
cp wildcard-cert-secret.yaml.template wildcard-cert-secret-new.yaml

# Edit the file and paste your new certificate and private key
# Replace PASTE_YOUR_CERTIFICATE_HERE and PASTE_YOUR_PRIVATE_KEY_HERE

# Encrypt with SOPS
sops --encrypt --in-place wildcard-cert-secret-new.yaml

# Replace the old secret
mv wildcard-cert-secret-new.yaml wildcard-cert-secret.yaml
```

**Option B: Edit Existing Secret**

```bash
cd talos-k8s/infrastructure/base/traefik

# Decrypt the existing secret
sops --decrypt wildcard-cert-secret.yaml > wildcard-cert-secret-plain.yaml

# Edit the plain file with your new certificate and key
# Update both tls.crt and tls.key sections

# Re-encrypt
sops --encrypt wildcard-cert-secret-plain.yaml > wildcard-cert-secret.yaml

# Remove the plain file
rm wildcard-cert-secret-plain.yaml
```

### Step 3: Deploy the Updated Certificate

```bash
git add wildcard-cert-secret.yaml
git commit -m "chore: renew wildcard TLS certificate for *.int.centreon.com"
git push

# Force reconciliation
flux reconcile kustomization infrastructure --with-source
```

### Step 4: Verify Certificate Update

```bash
# Check secret was updated
kubectl get secret wildcard-int-centreon-com-tls -n traefik -o yaml

# Restart Traefik pods to reload the certificate
kubectl rollout restart deployment traefik -n traefik

# Verify new certificate is in use
kubectl logs -n traefik -l app.kubernetes.io/name=traefik | grep -i tls
```

### Step 5: Test HTTPS Access

Access any service via HTTPS and verify the new certificate is being served:

```bash
# Check certificate details
curl -vI https://myapp.int.centreon.com 2>&1 | grep -A 10 "Server certificate"

# Or use openssl
openssl s_client -connect myapp.int.centreon.com:443 -servername myapp.int.centreon.com < /dev/null 2>&1 | openssl x509 -noout -dates
```

## Advanced: Using cert-manager for Automated Certificates

While the wildcard certificate covers most use cases, you can also configure cert-manager to automatically issue certificates for specific services.

### Example: Let's Encrypt ClusterIssuer

Create a ClusterIssuer for Let's Encrypt:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@centreon.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: traefik
```

### Example: Automated Certificate for Specific Service

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: default
spec:
  secretName: myapp-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.int.centreon.com
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: default
spec:
  tls:
    - hosts:
        - myapp.int.centreon.com
      secretName: myapp-tls  # Uses cert-manager issued certificate
  rules:
    - host: myapp.int.centreon.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

## Troubleshooting

### Check cert-manager Status

```bash
# Check all cert-manager components
kubectl get pods -n cert-manager

# Check logs
kubectl logs -n cert-manager -l app=cert-manager
kubectl logs -n cert-manager -l app=webhook
kubectl logs -n cert-manager -l app=cainjector
```

### Certificate Not Working

```bash
# Verify secret exists
kubectl get secret wildcard-int-centreon-com-tls -n traefik

# Check secret content (base64 encoded)
kubectl get secret wildcard-int-centreon-com-tls -n traefik -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text

# Check Traefik configuration
kubectl describe helmrelease traefik -n traefik
```

### SOPS Decryption Failing

```bash
# Verify SOPS age secret exists in cluster
kubectl get secret sops-age -n flux-system

# Check Flux logs for SOPS errors
kubectl logs -n flux-system deploy/kustomize-controller | grep -i sops

# Test local decryption
sops --decrypt infrastructure/base/traefik/wildcard-cert-secret.yaml
```

### Traefik Not Using Certificate

```bash
# Restart Traefik to reload configuration
kubectl rollout restart deployment traefik -n traefik

# Check Traefik logs
kubectl logs -n traefik -l app.kubernetes.io/name=traefik --tail=100

# Verify TLS store configuration
kubectl get tlsstore -A
```

## Monitoring

### Certificate Expiry

Set a reminder to renew the wildcard certificate before it expires. Typical certificate validity is 1 year.

**Add to calendar**: Renew `*.int.centreon.com` certificate 30 days before expiration.

### Verify Certificate Validity

```bash
# Check certificate expiration date
kubectl get secret wildcard-int-centreon-com-tls -n traefik -o json | \
  jq -r '.data."tls.crt"' | base64 -d | \
  openssl x509 -noout -enddate
```

### Prometheus Metrics

cert-manager exposes Prometheus metrics on port 9402:

- `certmanager_certificate_expiration_timestamp_seconds` - Certificate expiration time
- `certmanager_certificate_ready_status` - Certificate ready status

To enable ServiceMonitor for Prometheus:

```yaml
# In infrastructure/base/cert-manager/helmrelease.yaml
prometheus:
  enabled: true
  servicemonitor:
    enabled: true  # Enable when Prometheus is installed
```

## References

- **cert-manager Documentation**: https://cert-manager.io/docs/
- **Traefik TLS Documentation**: https://doc.traefik.io/traefik/https/tls/
- **SOPS Documentation**: https://github.com/mozilla/sops
- **Flux SOPS Integration**: https://fluxcd.io/flux/guides/mozilla-sops/
