# Model spec reference

The `ModelDeployment` custom resource is the entire deployment interface. Everything
the platform does is derived from this file.

```yaml
apiVersion: modelship.io/v1alpha1
kind: ModelDeployment
metadata:
  name: llama-3-2-3b-instruct   # DNS-1123 name, unique per namespace
spec:
  model:
    source: huggingface:meta-llama/Llama-3.2-3B-Instruct
    revision: <commit-sha>      # REQUIRED. Immutable pin.
  serving:
    backend: vllm               # vllm | llama.cpp
    quantization: awq           # none | awq | gguf-q4
    contextLength: 8192         # max sequence length
  resources:
    accelerator: nvidia         # nvidia | metal | cpu
    gpuClass: l4                # label selector for GPU class
    replicas: 2
    perReplica:
      gpu: 1
      memory: 16Gi
  rollout:
    strategy: canary            # canary | blue-green (later)
    canaryPercent: 10           # initial canary traffic weight
    evalSuite: evals/suites/llama-3-2-3b.yaml
    promotionThreshold: 0.95    # canary score / baseline score required to promote
  supplyChain:
    requireSigned: true         # Kyverno will reject unsigned artifacts
    maxTrivySeverity: HIGH      # CRITICAL always blocks; this sets the floor
  observability:
    tracing: true
    scrape: true
```

## Field semantics

### `model`
- `source` — `huggingface:<org>/<repo>` for now; other schemes (S3, MinIO) reserved.
- `revision` — mandatory commit SHA. The platform never deploys a moving tag.

### `serving.backend`
- `vllm` — NVIDIA path. Continuous batching, PagedAttention.
- `llama.cpp` — Apple Silicon (Metal) and CPU path for local development and
  CPU-only clusters. Quantization via `gguf-q4`.

The gateway is backend-agnostic: both expose the OpenAI-compatible
`/v1/chat/completions` surface.

### `rollout.promotionThreshold`
The eval harness computes `score(canary) / score(baseline)`. Promotion requires
`ratio ≥ promotionThreshold`. This is the heart of the platform: a model that
answers *differently but not worse* still promotes; a model that regresses
quality below the gate rolls back, even though it is perfectly "healthy".

### `supplyChain`
Enforced at admission by Kyverno, verified at CI time by the supply chain
pipeline. A mismatch between the two (e.g. spec demands signed, CI produced
unsigned) is a rollout blocker, not a warning.
