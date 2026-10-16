# evals/

The eval harness: the reason a "healthy" deployment can still be rejected.

## Suite format (`evals/suites/<model>.yaml`)

```yaml
name: llama-3-2-3b
dataset: datasets/golden-v1.jsonl
scorers:
  - type: exact-match        # deterministic
    weight: 0.4
  - type: llm-judge          # rubric-based, model-graded
    judgeModel: gpt-4o-mini  # or a local judge — ADR decision
    rubric: rubrics/default.md
    weight: 0.6
```

## Dataset format (`datasets/*.jsonl`)

```json
{"id": "qa-0001", "prompt": "...", "expected": "...", "category": "factual"}
```

Golden datasets are committed to the repo and versioned like code. They are the
contract the canary is held to.

## Scoring

- `exact-match`: deterministic comparison (normalized whitespace/case).
- `llm-judge`: rubric grading, temperature 0, output parsed to a 0–1 score;
  unparseable judge output is a harness error, not a zero.
- Suite score = weighted sum. `EvaluateGate` consumes
  `score(canary) / score(baseline)`.

## Rules

- Harness is a CLI (`python -m evals run --suite ... --endpoint ...`) so the
  Temporal activity just shells to it — no import coupling between the workflow
  and the harness.
- Judge calls are retried; flaky judge results are averaged over N runs, N ≥ 3.
- Every run emits a JSON report artifact: per-example scores, aggregate, config,
  git SHA of model + harness.
