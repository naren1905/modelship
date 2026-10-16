# Architecture

## Components

| Component | Responsibility | Lives in |
|-----------|----------------|----------|
| **Gateway** | OpenAI-compatible API: auth, rate limiting, traffic-split routing, tracing | `gateway/` |
| **Inference backend** | vLLM (NVIDIA) or llama.cpp (Metal/CPU) behind KServe or KubeRay | `platform/` |
| **ModelDeployment CRD** | Declarative model spec consumed by the rollout pipeline | `platform/crd/` |
| **Supply chain CI** | Build, scan, SBOM, sign — no unsigned artifact reaches the cluster | `.github/workflows/` |
| **ArgoCD** | GitOps sync of cluster state; the only writer of production state | `platform/argocd/` |
| **Kyverno** | Admission control: reject unsigned/unscanned images and manifests | `platform/kyverno/` |
| **Envoy Gateway** | Traffic splitting between baseline and canary | `platform/` |
| **Temporal workflows** | The rollout state machine: canary → eval → promote/rollback | `lifecycle/` |
| **Eval harness** | Scores canary vs baseline on golden datasets | `evals/` |
| **Observability** | OTel traces, Prometheus metrics, Grafana dashboards | `platform/`, `gateway/` |

## Data flows

### Push-time (deployment)

1. A model spec change is merged to `main`.
2. CI builds the serving image (BuildKit), pushes to the registry.
3. CI runs Trivy (fail on severity ≥ threshold), generates a Syft SBOM, signs with Cosign.
4. ArgoCD syncs the updated manifests.
5. Kyverno admits the workload only if signature and SBOM annotations verify.
6. The Temporal `RolloutWorkflow` starts: deploy canary at `canaryPercent` traffic,
   wait for readiness, run the eval suite, compare against `promotionThreshold`.
7. Pass → promote to 100%. Fail → automatic rollback. Either way, the decision
   and eval report are recorded.

### Request-time (serving)

```
client → gateway (auth, rate limit, [semantic cache]) → traffic split → backend
                │                                              │
                └── OTel trace context ──────────────► Prometheus / Grafana
```

## Security model

- **Supply chain:** unsigned images cannot be admitted (Kyverno + Cosign).
  SBOM is mandatory; Trivy severity gate is configurable per model spec.
- **Runtime:** gateway API keys, namespace-scoped RBAC, secrets via Vault or
  SealedSecrets (decided in an ADR).
- **Weights provenance:** model source is pinned to a commit SHA; the spec's
  `revision` is mandatory.

## Failure modes the platform must handle

| Failure | Response |
|---------|----------|
| Unhealthy canary | RolloutWorkflow rolls back before eval even runs |
| Eval regression | Auto-rollback, eval report attached to the workflow |
| Temporal server restart mid-rollout | Workflow state is durable, resumes automatically |
| Supply chain violation | Admission blocked; ArgoCD shows degraded sync |

## Decisions

Architecture decisions are recorded as [ADRs](adr/). Numbered, immutable, one
decision each. Start with [0001](adr/0001-record-architecture-decisions.md).
