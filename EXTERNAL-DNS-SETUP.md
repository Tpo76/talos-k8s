# ExternalDNS with Pi-hole Setup Guide

This document describes the ExternalDNS installation configured to automatically manage DNS records in Pi-hole for Kubernetes ingress resources.

## Overview

ExternalDNS automatically synchronizes exposed Kubernetes services and ingresses with DNS providers. In this setup, it's configured to manage Custom DNS records in Pi-hole.

### Architecture

- **Version**: 1.15.x (auto-updates for bugfixes)
- **Provider**: Pi-hole API v6
- **Pi-hole Server**: https://192.168.0.102:8443
- **Domain**: int.centreon.com
- **Sources**: Kubernetes Ingress resources
- **Policy**: upsert-only (doesn't delete manually managed records)

## Configuration

### Pi-hole Requirements

- **Pi-hole Version**: 6.0 or later (API v6)
- **API Access**: Password authentication enabled
- **Custom DNS**: ExternalDNS uses Pi-hole's Custom DNS (Local DNS) feature

### ExternalDNS Configuration

**Key Settings:**
- `--provider=pihole` - Use Pi-hole provider
- `--pihole-server=https://192.168.0.102:8443` - Pi-hole server URL
- `--pihole-api-version=6` - Use API v6
- `--registry=noop` - Disable TXT record registry (Pi-hole doesn't support TXT records)
- `--policy=upsert-only` - Only create/update records, never delete
- `--log-level=debug` - Enable debug logging

**Security:**
- TLS verification skipped for self-signed certificates
- Password stored as SOPS-encrypted secret
- Runs as non-root user (UID 65534)

## How It Works

### Automatic DNS Record Creation

When you create an Ingress resource with a hostname in the `int.centreon.com` domain, ExternalDNS automatically:

1. Detects the new Ingress
2. Extracts the hostname from spec.rules[].host
3. Gets the LoadBalancer IP from Traefik service (192.168.0.205)
4. Creates a Custom DNS record in Pi-hole: `<hostname> → 192.168.0.205`

### Example Workflow

```yaml
# Create an ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
    - host: myapp.int.centreon.com  # ExternalDNS picks this up
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

**Result:** ExternalDNS creates `myapp.int.centreon.com → 192.168.0.205` in Pi-hole

## Usage

### Verify ExternalDNS is Running

```bash
# Check pod status
kubectl get pods -n external-dns

# Check logs
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns

# Watch for DNS record creation
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns --follow
```

### Test with the nginx Example

The nginx ingress (`nginx.int.centreon.com`) should automatically get a DNS record:

```bash
# Check if ExternalDNS created the record
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns | grep nginx.int.centreon.com

# Test DNS resolution
nslookup nginx.int.centreon.com 192.168.0.102

# Or use dig
dig @192.168.0.102 nginx.int.centreon.com
```

### Verify Records in Pi-hole

1. Access Pi-hole admin: https://192.168.0.102:8443/admin/
2. Login with password: `centreon`
3. Navigate to **Local DNS → DNS Records**
4. Look for records created by ExternalDNS

### Creating New Ingresses

ExternalDNS will automatically manage any Ingress in the `int.centreon.com` domain:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
spec:
  tls:
    - hosts:
        - grafana.int.centreon.com
  rules:
    - host: grafana.int.centreon.com  # Automatically gets DNS record
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 3000
```

After creating this ingress:
- ExternalDNS creates: `grafana.int.centreon.com → 192.168.0.205`
- You can immediately access: https://grafana.int.centreon.com
- Certificate is automatically provided by Traefik (wildcard cert)

## Advanced Configuration

### Filtering by Annotation

To explicitly control which ingresses ExternalDNS manages, you can add annotation filters:

Edit `infrastructure/base/external-dns/helmrelease.yaml`:

```yaml
extraArgs:
  - --annotation-filter=external-dns.alpha.kubernetes.io/hostname
```

Then only ingresses with this annotation will be managed:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    external-dns.alpha.kubernetes.io/hostname: myapp.int.centreon.com
spec:
  # ...
```

### Managing Service LoadBalancers

You can also manage DNS for LoadBalancer services:

Edit `infrastructure/base/external-dns/helmrelease.yaml`:

```yaml
sources:
  - ingress
  - service  # Add service source
```

Then annotate services:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    external-dns.alpha.kubernetes.io/hostname: myapp-direct.int.centreon.com
spec:
  type: LoadBalancer
  # ...
```

### Multiple Domains

To manage multiple domains, update the domain filter:

```yaml
domainFilters:
  - int.centreon.com
  - lab.centreon.com
```

## Troubleshooting

### Check ExternalDNS Status

```bash
# View logs
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns --tail=100

# Check for errors
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns | grep -i error

# Describe the deployment
kubectl describe deployment external-dns -n external-dns

# Check the HelmRelease
kubectl get helmrelease -n external-dns
```

### Common Issues

**DNS records not being created:**

1. Check ExternalDNS logs for errors:
   ```bash
   kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns
   ```

2. Verify Pi-hole credentials:
   ```bash
   kubectl get secret pihole-password -n external-dns -o yaml
   ```

3. Check Ingress is in the correct domain:
   ```bash
   kubectl get ingress -A -o custom-columns=NAME:.metadata.name,HOST:.spec.rules[0].host
   ```

**Pi-hole API authentication failing:**

```bash
# Test Pi-hole API access manually
curl -k https://192.168.0.102:8443/admin/api.php?auth=<password_hash>

# Check if password secret is correct
kubectl get secret pihole-password -n external-dns -o jsonpath='{.data.EXTERNAL_DNS_PIHOLE_PASSWORD}' | base64 -d
```

**TLS certificate errors:**

ExternalDNS is configured to skip TLS verification (`EXTERNAL_DNS_PIHOLE_TLS_SKIP_VERIFY=true`). If you want to use proper certificates:

1. Remove or set to `false` in `helmrelease.yaml`
2. Ensure Pi-hole has a valid certificate trusted by the system

### Debug Mode

ExternalDNS is already configured with `--log-level=debug`. To see even more details:

```bash
# Watch logs in real-time
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns -f

# Filter for specific hostname
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns | grep "myapp.int.centreon.com"
```

### Manual DNS Record Management

With `--policy=upsert-only`, ExternalDNS won't delete manually created records in Pi-hole. This is useful if you have:
- Static DNS entries you want to keep
- Records for services outside Kubernetes
- Override entries for specific hosts

To change this behavior to allow deletions, edit `helmrelease.yaml`:

```yaml
extraArgs:
  - --policy=sync  # WARNING: Will delete records not managed by ExternalDNS
```

## Updating Pi-hole Password

When the Pi-hole password changes:

```bash
cd talos-k8s/infrastructure/base/external-dns

# Decrypt the secret
sops --decrypt pihole-secret.yaml > pihole-secret-plain.yaml

# Edit and update the password
# Change the EXTERNAL_DNS_PIHOLE_PASSWORD value

# Re-encrypt
sops --encrypt pihole-secret-plain.yaml > pihole-secret.yaml

# Clean up
rm pihole-secret-plain.yaml

# Commit and push
git add pihole-secret.yaml
git commit -m "chore: update Pi-hole password"
git push

# Reconcile
flux reconcile kustomization infrastructure --with-source
```

## Monitoring

### Key Metrics to Watch

Check logs for these indicators:

- **Success**: `msg="Applying changes"`
- **Skipping**: `msg="Skipping update, no changes detected"`
- **Errors**: `msg="Failed to update records"`

### Example Successful Update

```
time="..." level=info msg="Desired change: CREATE myapp.int.centreon.com A [Id: ]"
time="..." level=info msg="Applying changes"
time="..." level=info msg="Record created successfully" hostname="myapp.int.centreon.com"
```

## Best Practices

1. **Use upsert-only policy** - Prevents accidental deletion of manual DNS entries
2. **Filter by domain** - Only manage records in your Kubernetes domain (int.centreon.com)
3. **Monitor logs** - Watch for errors and successful updates
4. **Test DNS resolution** - Always verify records are created correctly
5. **Document manual entries** - Keep track of any manual Pi-hole records you create

## References

- **ExternalDNS Documentation**: https://kubernetes-sigs.github.io/external-dns/
- **Pi-hole Provider Guide**: https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/pihole/
- **ExternalDNS GitHub**: https://github.com/kubernetes-sigs/external-dns
- **Pi-hole Documentation**: https://docs.pi-hole.net/

## Sources

Configuration based on official ExternalDNS Pi-hole provider documentation:
- [Pi-hole for Kubernetes external-dns](https://technologistcreative.hashnode.dev/using-pi-hole-as-your-external-dns-provider-in-kubernetes)
- [ExternalDNS Pi-hole Tutorial](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/pihole.md)
- [Setting up ExternalDNS for Pi-hole](https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/pihole/)
