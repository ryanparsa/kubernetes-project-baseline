# Kubernetes Project Baseline

[![Test with kind](https://github.com/ryanparsa/kubernetes-namespace-template/actions/workflows/test.yaml/badge.svg)](https://github.com/ryanparsa/kubernetes-namespace-template/actions/workflows/test.yaml)

A production-ready baseline for deploying a service on Kubernetes. Replace every occurrence of `project-n` with your service name, then apply the manifests.

## Included Resources

> **Pure Kubernetes by default.** 13 of 16 resources use only built-in Kubernetes APIs.
> Three resources need external components: `hpa.yaml` (Metrics Server), `vpa.yaml` (VPA CRDs), and `gateway.yaml`/`httproute.yaml` (Gateway API CRDs + a controller).
> You can delete any of those files if you don't need them — the rest applies cleanly to any standard cluster.

| File | Resource(s) | Purpose | Requires |
|------|-------------|---------|----------|
| `namespace.yaml` | Namespace | Creates the namespace with Pod Security Admission `restricted` enforcement | — |
| `serviceaccount.yaml` | ServiceAccount | Workload identity with token auto-mount disabled | — |
| `configmap.yaml` | ConfigMap | Non-sensitive configuration loaded as environment variables | — |
| `secret.yaml` | Secret (x2) | App credentials (file-mounted) + TLS certificate | — |
| `resourcequota.yaml` | ResourceQuota | Namespace-level CPU/memory/pod caps | — |
| `limitrange.yaml` | LimitRange | Per-container default requests and limits | — |
| `networkpolicy.yaml` | NetworkPolicy (x3) | Default-deny all, allow gateway ingress, allow DNS egress | — |
| `deployment.yaml` | Deployment | Main workload - non-root, read-only FS, rolling update | — |
| `service.yaml` | Service | ClusterIP exposing the deployment on port 80 | — |
| `hpa.yaml` | HorizontalPodAutoscaler | Scales 3-10 replicas on CPU/memory (50% target) | [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) |
| `vpa.yaml` | VerticalPodAutoscaler | Advisory right-sizing recommendations (mode: Off) | [VPA CRDs](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) |
| `pdb.yaml` | PodDisruptionBudget | Maintains at least 2 pods during voluntary disruptions | — |
| `role.yaml` | Role | Empty RBAC Role - add rules here as your workload requires | — |
| `rolebinding.yaml` | RoleBinding | Binds the Role to the workload ServiceAccount | — |
| `gateway.yaml` | Gateway | Gateway with HTTP (80) and HTTPS (443) listeners | [Gateway API CRDs](https://gateway-api.sigs.k8s.io/) + [controller](https://gateway-api.sigs.k8s.io/implementations/) |
| `httproute.yaml` | HTTPRoute (x2) | HTTP->HTTPS redirect (301) + HTTPS backend routing | [Gateway API CRDs](https://gateway-api.sigs.k8s.io/) + [controller](https://gateway-api.sigs.k8s.io/implementations/) |

## Optional Add-ons

> None of these are required. They complement the baseline depending on your needs.

| Tool | Notes |
|------------|-------|
| [cert-manager](https://cert-manager.io/) | Automates TLS certificate provisioning instead of manual `project-n-tls` secret |
| [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) | Encrypts `project-n-secret` for safe GitOps storage |
| [External Secrets](https://external-secrets.io/) | Syncs secrets from external vaults (AWS, GCP, Vault, etc.) |
| [Secrets Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) | Mounts secrets from external vaults directly as volumes - alternative to External Secrets |
| [External DNS](https://github.com/kubernetes-sigs/external-dns) | Automatically creates DNS records for the Gateway hostname |
| [Karpenter](https://github.com/kubernetes-sigs/karpenter) | Node autoscaler - the `safe-to-evict` annotation on the pod template is compatible with it |
| [Descheduler](https://github.com/kubernetes-sigs/descheduler) | Rebalances pods across nodes - complements the topology spread constraint |
| [Security Profiles Operator](https://github.com/kubernetes-sigs/security-profiles-operator) | Manages seccomp and AppArmor profiles - extends the `seccompProfile: RuntimeDefault` used here |

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
- **Read-only root filesystem** - writable paths (e.g. `/tmp`) use size-limited `emptyDir` volumes
- **Capabilities** - all Linux capabilities dropped; `seccompProfile: RuntimeDefault`
- **NetworkPolicy** - default-deny ingress/egress; explicit rules for gateway and DNS only
- **Secrets** - mounted as files under `/etc/secret`, not environment variables
- **RBAC** - empty Role with RoleBinding; grants zero permissions by default (principle of least privilege)

### High Availability
- **HPA** - horizontal scaling 3-10 replicas based on CPU/memory utilization
- **VPA** - advisory mode alongside HPA (no interference)
- **PDB** - `minAvailable: 2` allows one pod to be disrupted at a time
- **Topology spread** - pods spread evenly across nodes (`maxSkew: 1`)
- **Rolling update** - `maxUnavailable: 0, maxSurge: 1` for zero-downtime deploys
- **Graceful shutdown** - `preStop` sleep drains in-flight requests before SIGTERM; `terminationGracePeriodSeconds: 30`
- **Startup probe** - gates liveness/readiness checks until the container has passed initialization (up to 60s)

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

## Security Compliance

| Control | Implementation | Standard |
|---------|---------------|----------|
| Insecure workload configurations | PSA `restricted`, non-root, read-only FS, drop ALL caps, seccomp | OWASP K01, CIS 5.2 |
| Secrets management | File-mounted secrets, no env vars, etcd encryption recommended | OWASP K03, CIS 5.4 |
| Network segmentation | Default-deny NetworkPolicy; allow only gateway + DNS | OWASP K05, CIS 5.3 |
| Least privilege | Empty RBAC Role, `automountServiceAccountToken: false` | OWASP K09, CIS 5.1 |
| Resource limits | ResourceQuota + LimitRange on every container | CIS 5.6 |
| Availability | HPA (3-10), PDB (minAvailable: 2), topology spread, zero-downtime rolling update | CIS 5.7 |

Controls not enforced at the manifest level (require cluster-level configuration): etcd encryption at rest (CIS 1.2.34), audit logging (CIS 1.2.22), API server hardening (CIS 1.2.*).

## Testing

CI runs on every push and pull request using a local [kind](https://kind.sigs.k8s.io/) cluster:

```
namespace -> serviceaccount -> role -> rolebinding -> configmap -> secret -> resourcequota -> limitrange
-> networkpolicy -> deployment -> service -> hpa -> vpa -> pdb -> gateway -> httproute
```


## Related Projects

- [klist](https://github.com/ryanparsa/klist) - Interactive Kubernetes operational checklist to verify cluster readiness
- [kubernetes-certification](https://github.com/ryanparsa/kubernetes-certification) - CKA/KCNA/KCSA certification training scenarios

