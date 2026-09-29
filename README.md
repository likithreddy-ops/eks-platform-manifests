# eks-platform-manifests

Kubernetes manifests (packaged as a Helm chart) for a microservices voting application — deployed onto the EKS cluster provisioned in [eks-platform-infra](https://github.com/likithreddy-ops/eks-platform-infra), via GitOps ([eks-platform-gitops](https://github.com/likithreddy-ops/eks-platform-gitops)).

Demonstrates production-pattern platform engineering practices: NetworkPolicy segmentation, least-privilege RBAC, liveness/readiness probes, resource limits, PodDisruptionBudgets, pod anti-affinity across Availability Zones, and Horizontal Pod Autoscaling — all verified working end-to-end on real AWS infrastructure, not just minikube.

---

## Architecture

Five services, three tiers:

| Service | Tier | Role |
|---|---|---|
| **Vote** | frontend | Public-facing voting UI (2 replicas) |
| **Redis** | backend | In-memory vote queue (1 replica) |
| **Worker** | backend | Reads votes from Redis, writes to Postgres (1 replica, no exposed port) |
| **Postgres** (`db`) | database | Persistent vote storage (StatefulSet, 1 replica, EBS-backed PVC) |
| **Result** | frontend | Public-facing results UI (2 replicas) |

**Security:** NetworkPolicy restricts Redis to ingress only from Vote + Worker, and Postgres to ingress only from Worker + Result — Vote is structurally blocked from reaching the database directly. A dedicated ServiceAccount/Role/RoleBinding demonstrates least-privilege RBAC.

**Reliability:** every workload has liveness/readiness probes and resource requests/limits; Vote and Result have PodDisruptionBudgets (`minAvailable: 1`); Vote has an HPA (CPU-based, 2–3 replicas).

## Stack

Helm · Kubernetes (Deployments, StatefulSet, Services, NetworkPolicy, RBAC, HPA, PDB) · [Docker's Example Voting App](https://github.com/dockersamples/example-voting-app) container images

## Usage

Deployed via ArgoCD (see [eks-platform-gitops](https://github.com/likithreddy-ops/eks-platform-gitops)) — not manual `kubectl apply`. For local testing:

```bash
helm install voter-app .
```

## Real Issues Hit & Fixed

Genuine debugging, not a copy-paste chart — details in `docs/implementation-notes.md`:

1. **Worker's `exec` liveness probe caused a false-positive crash loop.** The minimal `dockersamples/examplevotingapp_worker` image lacks standard process-inspection tools (`pgrep`, then `ps`, both missing) — confirmed Worker was actually healthy via its own logs before removing the probe entirely and keeping only resource limits.
2. **Ingress unreachable on minikube (Windows), but not a real bug.** Diagnosed to a minikube/Windows-specific `NodePort` networking limitation via `kubectl port-forward`, which worked immediately — confirmed the app/Service layer was correct, and the issue doesn't recur on real EKS (which provisions a genuine AWS Load Balancer).
3. **Postgres crash-looping on real EKS (`initdb: directory ... is not empty`).** AWS EBS auto-creates a `lost+found` directory at the volume root; `initdb` refuses to treat that as empty. Fixed with `PGDATA=/var/lib/postgresql/data/pgdata`, matching the official reference manifests — required deleting and letting ArgoCD recreate the StatefulSet (Kubernetes forbids most in-place spec edits) plus a fresh PVC.
4. **Real EKS has no default StorageClass or EBS CSI driver** (unlike minikube) — required a Terraform-side fix (see [eks-platform-infra](https://github.com/likithreddy-ops/eks-platform-infra)), not something fixable from the manifests alone.

## Verified End-to-End

Full pipeline confirmed working on real AWS: a vote cast through Vote's UI correctly appears on Result's page, proving Vote → Redis → Worker → Postgres → Result functions correctly together — deployed declaratively via ArgoCD, not manual commands.

## Deferred / Future Enhancements

- Self-service preview environments (PR-triggered temporary namespace + auto-teardown)
- Dedicated namespace (currently deployed to `default` for simplicity — a real multi-tenant cluster would isolate this)
- Service mesh (Istio/Linkerd), distributed tracing, API Gateway pattern
- Sealed Secrets / External Secrets Operator (Postgres credentials are currently plain env vars — acceptable for this demo's scope, documented as a known simplification)
- Chaos testing (kill a node, not just a pod)
