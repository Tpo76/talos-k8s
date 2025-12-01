# Traefik Ingress Controller Setup

This document describes the Traefik ingress controller installation for the Talos Kubernetes cluster.

## Overview

Traefik is installed as the default ingress controller for the cluster, providing HTTP/HTTPS routing to services.

### Architecture

- **Version**: v32.1.1 (Traefik v3.x)
- **Deployment**: 2 replicas for high availability
- **Service Type**: LoadBalancer (via MetalLB)
- **Ingress Class**: `traefik` (set as default)
- **Monitoring**: Prometheus metrics enabled on port 9100

## Configuration

### Service Exposure

Traefik is exposed via MetalLB LoadBalancer with the following ports:

- **HTTP**: Port 80 (redirectable to HTTPS)
- **HTTPS**: Port 443 with HTTP/3 (QUIC) support
- **Dashboard**: Port 9000 (internal only, not exposed externally)
- **Metrics**: Port 9100 (internal only, for Prometheus)

MetalLB assigns an IP from the pool (192.168.0.200-250). Check the assigned IP:

```bash
kubectl get svc -n traefik traefik
```

### Ingress Class

Traefik is configured as the **default ingress controller** via the `IngressClass` resource. This means:

- All `Ingress` resources without an explicit `ingressClassName` will use Traefik
- You can still specify other ingress controllers if needed by setting `ingressClassName`

### High Availability

- **Replicas**: 2 pods for redundancy
- **Pod Disruption Budget**: Minimum 1 pod always available
- **Anti-affinity**: Pods spread across different nodes when possible
- **External Traffic Policy**: `Local` (preserves source IP addresses)

### Security

- Runs as non-root user (UID 65532)
- Read-only root filesystem
- Drops all Linux capabilities
- Resource limits enforced (CPU: 1000m, Memory: 512Mi)

## Usage

### Creating an Ingress Resource

**Simple HTTP Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: default
spec:
  rules:
    - host: myapp.example.com
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

**HTTPS Ingress with TLS:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-tls
  namespace: default
spec:
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls-cert  # TLS certificate secret
  rules:
    - host: myapp.example.com
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

### Advanced Routing with IngressRoute (CRD)

Traefik also supports its native `IngressRoute` CRD for advanced features:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: my-app-route
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`myapp.example.com`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: my-app-service
          port: 80
      middlewares:
        - name: rate-limit
  tls:
    secretName: myapp-tls-cert
```

### Common Middleware Examples

**HTTP to HTTPS Redirect:**

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: https-redirect
  namespace: default
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

**Basic Authentication:**

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: default
spec:
  basicAuth:
    secret: auth-secret  # Create with: htpasswd -c auth username
```

**Rate Limiting:**

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: default
spec:
  rateLimit:
    average: 100
    burst: 50
```

## Accessing the Dashboard

The Traefik dashboard is enabled but not exposed externally for security.

**Access via port-forward:**

```bash
kubectl port-forward -n traefik svc/traefik 9000:9000
```

Then open: http://localhost:9000/dashboard/

**Access via Ingress (optional):**

Create an ingress with basic authentication:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-dashboard
  namespace: traefik
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: traefik-dashboard-auth@kubernetescrd
spec:
  rules:
    - host: traefik.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: traefik
                port:
                  number: 9000
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: dashboard-auth
  namespace: traefik
spec:
  basicAuth:
    secret: traefik-dashboard-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: traefik-dashboard-auth
  namespace: traefik
type: Opaque
stringData:
  # Generate with: htpasswd -nb admin password | base64
  users: |
    admin:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/
```

## Monitoring

### Prometheus Metrics

Traefik exposes Prometheus metrics on port 9100 (internal only).

**Scrape configuration for Prometheus:**

```yaml
- job_name: 'traefik'
  kubernetes_sd_configs:
    - role: pod
      namespaces:
        names:
          - traefik
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
```

**Key metrics:**

- `traefik_entrypoint_requests_total` - Total requests per entrypoint
- `traefik_entrypoint_request_duration_seconds` - Request duration
- `traefik_router_requests_total` - Requests per router
- `traefik_service_requests_total` - Requests per service

### Access Logs

Access logs are enabled in JSON format. View them with:

```bash
kubectl logs -n traefik -l app.kubernetes.io/name=traefik --tail=100 -f
```

## TLS / HTTPS Configuration

### Option 1: Manual Certificate Management

Create a TLS secret with your certificate:

```bash
kubectl create secret tls myapp-tls-cert \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key \
  -n default
```

Reference it in your Ingress (see examples above).

### Option 2: Cert-Manager Integration (Recommended)

Install cert-manager and configure automatic certificate issuance:

1. **Install cert-manager** (via Helm or FluxCD)
2. **Create ClusterIssuer** for Let's Encrypt:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@yourdomain.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: traefik
```

3. **Use in Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls-cert  # cert-manager creates this automatically
  rules:
    - host: myapp.example.com
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

## Troubleshooting

### Check Traefik Status

```bash
# Check pods
kubectl get pods -n traefik

# Check service and LoadBalancer IP
kubectl get svc -n traefik

# Check logs
kubectl logs -n traefik -l app.kubernetes.io/name=traefik

# Check Helm release status
kubectl get helmrelease -n traefik
```

### Common Issues

**LoadBalancer IP pending:**

```bash
# Check MetalLB status
kubectl get pods -n metallb-system
kubectl get ipaddresspool,l2advertisement -n metallb-system

# Check service events
kubectl describe svc traefik -n traefik
```

**Ingress not routing traffic:**

```bash
# Verify ingress exists
kubectl get ingress -A

# Check ingress details
kubectl describe ingress <name> -n <namespace>

# Check Traefik routing configuration via dashboard
kubectl port-forward -n traefik svc/traefik 9000:9000
# Open: http://localhost:9000/dashboard/
```

**Certificate issues:**

```bash
# Check TLS secret exists
kubectl get secret <tls-secret-name> -n <namespace>

# Check cert-manager certificate status (if using cert-manager)
kubectl get certificate -A
kubectl describe certificate <cert-name> -n <namespace>
```

## GitOps Deployment

Traefik is deployed via FluxCD and follows the GitOps workflow:

1. **Configuration files**: `infrastructure/base/traefik/`
2. **Make changes** to the configuration files
3. **Commit and push** to Git
4. **Flux reconciles** automatically (within 10 minutes)

**Force immediate reconciliation:**

```bash
flux reconcile kustomization infrastructure
```

**Check reconciliation status:**

```bash
flux get helmrelease -n traefik
kubectl get helmrelease traefik -n traefik -o yaml
```

## Configuration Customization

### Updating Traefik Configuration

Edit `infrastructure/base/traefik/helmrelease.yaml` to customize:

- Replica count
- Resource limits
- Service type or annotations
- TLS settings
- Middleware configurations
- Access log settings
- Metrics configuration

After editing, commit and push. Flux will apply changes automatically.

### Environment-Specific Overrides

For staging-specific configurations, create an overlay in `infrastructure/staging/traefik/`:

```yaml
# infrastructure/staging/traefik/helmrelease-patch.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: traefik
  namespace: traefik
spec:
  values:
    deployment:
      replicas: 1  # Single replica for staging
```

Add to `infrastructure/staging/kustomization.yaml`:

```yaml
resources:
  - ../base

patchesStrategicMerge:
  - traefik/helmrelease-patch.yaml
```

## References

- **Traefik Documentation**: https://doc.traefik.io/traefik/
- **Traefik Helm Chart**: https://github.com/traefik/traefik-helm-chart
- **Kubernetes Ingress**: https://kubernetes.io/docs/concepts/services-networking/ingress/
- **MetalLB**: https://metallb.universe.tf/
- **Cert-Manager**: https://cert-manager.io/
