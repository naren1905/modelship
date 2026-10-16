# platform/

Everything that runs *on* the cluster or defines cluster state.

```
platform/
├── kind/      dev cluster config (kind + metrics-server + gateway port mappings)
├── argocd/    app-of-apps: the platform bootstraps itself from this directory
├── kyverno/   ClusterPolicies: verifyImage (Cosign), require SBOM annotation
├── temporal/  Temporal server / namespace manifests (dev: temporalite)
└── crd/       ModelDeployment CRD + (your call) controller, or Kustomize-only
```

## Conventions

- Everything ArgoCD manages is declarative in this repo. No `kubectl apply` by hand
  to anything ArgoCD owns.
- Local dev target: Apple Silicon (Metal). Production path: NVIDIA + GPU Operator.
- `make` targets (added in Phase 1) should make every entry point one command:
  `make cluster`, `make deploy-platform`, `make smoke`.

## Open decisions (each needs an ADR)

- KServe vs KubeRay as the serving substrate → see issues
- CRD with a custom controller vs Kustomize overlays + external rollout logic
- Secrets: Vault vs SealedSecrets for the public/demo story
