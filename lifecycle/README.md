# lifecycle/

Temporal workflows that own the model rollout state machine. This is the
component that makes deployments *safe* instead of just automated.

## RolloutWorkflow

```
DeployCanary ──► WaitForHealthy ──► RunEvalSuite ──► EvaluateGate ──► Promote
      │                │                  │               │
      └──► Rollback ◄──┴──────────────────┴───────────────┘   (score < threshold)
```

### Activities (all idempotent, all with explicit retry policies)

| Activity | Responsibility |
|----------|----------------|
| `DeployCanary` | Create/patch canary deployment at `spec.rollout.canaryPercent` |
| `WaitForHealthy` | Poll readiness; timeout + failure → Rollback |
| `RunEvalSuite` | Execute the eval harness against canary and baseline, produce report |
| `EvaluateGate` | `score(canary)/score(baseline) ≥ promotionThreshold` → bool |
| `Promote` | Scale canary to 100%, retire baseline |
| `Rollback` | Remove canary, restore baseline — the compensation activity |

### Workflow requirements

- **Durable:** a Temporal server restart mid-rollout must resume, not restart.
  This is demo-able and goes in the README GIF.
- **Signals:** `pause` (freeze traffic split), `abort` (force rollback).
- **Query:** `status` returns phase, eval scores, elapsed time — no history reload.
- The eval report is a workflow-attached artifact; the promote/rollback decision
  must always be auditable after the fact.

## Tech

- Python SDK (`temporalio`). Dev server: temporalite. Prod: `platform/temporal/`.
- Workflow code is versioned: never edit a running workflow's logic; new version,
  new rollout.
