# Kubernetes Project Baseline

[![Test with kind](https://github.com/ryanparsa/kubernetes-namespace-template/actions/workflows/test.yaml/badge.svg)](https://github.com/ryanparsa/kubernetes-namespace-template/actions/workflows/test.yaml)

A production-ready baseline for deploying a service on Kubernetes. Replace every occurrence of `project-n` with your service name, then apply the manifests.

## Included Resources

| File | Resource(s) | Purpose |
|------|-------------|---------|
| `namespace.yaml` | Namespace | Creates the namespace with Pod Security Admission `restricted` enforcement |
| `serviceaccount.yaml` | ServiceAccount | Workload identity with token auto-mount disabled |
| `configmap.yaml` | ConfigMap | Non-sensitive configuration loaded as environment variables |
| `secret.yaml` | Secret (x2) | App credentials (file-mounted) + TLS certificate |
| `resourcequota.yaml` | ResourceQuota | Namespace-level CPU/memory/pod caps |
| `limitrange.yaml` | LimitRange | Per-container default requests and limits |
| `networkpolicy.yaml` | NetworkPolicy (x3) | Default-deny all, allow gateway ingress, allow DNS egress |
| `deployment.yaml` | Deployment | Main workload - non-root, read-only FS, rolling update |
| `service.yaml` | Service | ClusterIP exposing the deployment on port 80 |
| `hpa.yaml` | HorizontalPodAutoscaler | Scales 3-10 replicas on CPU/memory (50% target) |
| `vpa.yaml` | VerticalPodAutoscaler | Advisory right-sizing recommendations (mode: Off) |
| `pdb.yaml` | PodDisruptionBudget | Maintains at least 2 pods during voluntary disruptions |
| `gateway.yaml` | Gateway | Gateway with HTTP (80) and HTTPS (443) listeners |
| `httproute.yaml` | HTTPRoute (x2) | HTTP->HTTPS redirect (301) + HTTPS backend routing |

## Prerequisites

| Type | Dependency | Notes |
|------|------------|-------|
| Required CRD | [Gateway API](https://gateway-api.sigs.k8s.io/) | Needed for Gateway and HTTPRoute resources |
| Required CRD | [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) | Needed for VPA resource |
| Gateway controller (pick one) | [Traefik](https://traefik.io/traefik/) . [Envoy Gateway](https://gateway.envoyproxy.io/) . [NGINX Gateway Fabric](https://github.com/nginxinc/nginx-gateway-fabric) . [Cilium](https://cilium.io/) | Set `gatewayClassName` in `gateway.yaml` to match your choice |
| Required | [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) | Needed for HPA to read CPU/memory utilization |
| Recommended | [cert-manager](https://cert-manager.io/) | Automates TLS certificate provisioning instead of manual `project-n-tls` secret |
| Recommended | [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) | Encrypts secrets for safe GitOps storage |
| Recommended | [External Secrets](https://external-secrets.io/) | Syncs secrets from external vaults (AWS, GCP, Vault, etc.) |

## Usage

```bash
# 1. Clone
git clone https://github.com/ryanparsa/kubernetes-namespace-template.git my-service
cd my-service

# 2. Replace the placeholder name
find . -name '*.yaml' | xargs sed -i 's/project-n/my-service/g'

# 3. Edit placeholder values
#    - deployment.yaml  -> image, resource limits
#    - gateway.yaml     -> hostname (project-n.example.com)
#    - secret.yaml      -> credentials, TLS cert/key
#    - hpa.yaml         -> min/max replica counts

# 4. Apply
kubectl apply -f .
```

## Architecture

### Security
- **Pod Security Admission** - `restricted` profile enforced at namespace level
- **Non-root** - runs as UID 1000, `runAsNonRoot: true`
- **Read-only root filesystem** - writable paths (`/tmp`, `/var/cache/nginx`, `/var/run`) use size-limited `emptyDir` volumes
- **Capabilities** - all Linux capabilities dropped; `seccompProfile: RuntimeDefault`
- **NetworkPolicy** - default-deny ingress/egress; explicit rules for gateway and DNS only
- **Secrets** - mounted as files under `/etc/secret`, not environment variables

### High Availability
- **HPA** - horizontal scaling 3-10 replicas based on CPU/memory utilization
- **VPA** - advisory mode alongside HPA (no interference)
- **PDB** - `minAvailable: 2` allows one pod to be disrupted at a time
- **Topology spread** - pods spread evenly across nodes (`maxSkew: 1`)
- **Rolling update** - `maxUnavailable: 0, maxSurge: 1` for zero-downtime deploys

### Networking
- **Kubernetes Gateway API** - modern replacement for Ingress; bring your own Gateway controller and set `gatewayClassName` accordingly
- **HTTP->HTTPS redirect** - all port-80 traffic receives a 301 redirect
- **TLS termination** - handled at the Gateway using the `project-n-tls` secret

## Labeling Convention

All resources use `app.kubernetes.io/` semantic labels:

```yaml
app.kubernetes.io/name: project-n
app.kubernetes.io/component: server        # or gateway, autoscaler
app.kubernetes.io/part-of: project-n
app.kubernetes.io/managed-by: kubectl
```

## Testing

CI runs on every push and pull request using a local [kind](https://kind.sigs.k8s.io/) cluster. Each manifest is applied and verified independently:

```
namespace -> serviceaccount -> configmap -> secret -> resourcequota -> limitrange
-> networkpolicy -> deployment -> service -> hpa -> vpa -> pdb -> gateway -> httproute
```
