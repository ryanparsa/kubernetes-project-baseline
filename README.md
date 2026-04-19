# Kubernetes Project Baseline

A production-ready baseline for deploying a service on Kubernetes. Replace every occurrence of `project-n` with your service name, then apply the manifests.

[![Test with kind](https://github.com/ryanparsa/kubernetes-project-baseline/actions/workflows/test.yaml/badge.svg)](https://github.com/ryanparsa/kubernetes-project-baseline/actions/workflows/test.yaml)

## Files

| File | Kind | Purpose |
|---|---|---|
| `namespace.yaml` | Namespace | Creates the namespace with Pod Security Admission labels enforcing the `restricted` profile |
| `serviceaccount.yaml` | ServiceAccount | Dedicated identity for the workload; auto-mount disabled |
| `resourcequota.yaml` | ResourceQuota | Caps total CPU/memory requests and limits across all pods in the namespace |
| `limitrange.yaml` | LimitRange | Injects default CPU/memory requests and limits for containers that don't set their own |
| `networkpolicy.yaml` | NetworkPolicy (×3) | Default-deny all traffic; allow ingress from the Traefik Gateway controller; allow egress to kube-dns |
| `deployment.yaml` | Deployment | Rolling-update deployment with security context, probes, topology spread, and emptyDir volumes for nginx writable dirs |
| `service.yaml` | Service | ClusterIP service; only routes to ready pods |
| `hpa.yaml` | HorizontalPodAutoscaler | Scales 3–10 replicas on CPU and memory (50% target utilization); 5-minute scale-down stabilisation window |
| `vpa.yaml` | VerticalPodAutoscaler | Runs in `Off` mode - recommendations only, no automatic resizing |
| `pdb.yaml` | PodDisruptionBudget | Keeps at least 2 pods available during voluntary disruptions |
| `gateway.yaml` | Gateway | Traefik Gateway API listeners on port 80 (HTTP) and 443 (HTTPS/TLS termination) scoped to `project-n.example.com` |
| `httproute.yaml` | HTTPRoute (×2) | HTTP → HTTPS redirect (301) and HTTPS → backend routing, both scoped to `project-n.example.com` |

## Usage

1. **Clone** this repo and `cd` into it.
2. **Replace placeholders** - swap `project-n` with your service name and `project-n.example.com` with your real hostname throughout all files:
   ```sh
   find . -name '*.yaml' | xargs sed -i '' 's/project-n/your-service/g'
   find . -name '*.yaml' | xargs sed -i '' 's/project-n\.example\.com/your-service.example.com/g'
   ```
3. **Provision the TLS secret** referenced in `gateway.yaml` (`project-n-tls`) via cert-manager or manually before applying the Gateway.
4. **Apply:**
   ```sh
   kubectl apply -f namespace.yaml
   kubectl apply -f .
   ```

## Requirements

- Kubernetes 1.25+ (for Pod Security Admission)
- [Traefik](https://doc.traefik.io/traefik/providers/kubernetes-gateway/) installed in the `traefik` namespace with the Gateway API provider enabled
- Gateway API CRDs installed (`gateway.networking.k8s.io/v1`)
- VPA CRDs installed if you want VPA recommendations (`vpa.yaml`)
- Metrics Server installed for HPA to function (`hpa.yaml`)
