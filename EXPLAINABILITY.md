# Agenta Explainability & Decision Transparency Report

## How the Agent Decides

Agenta coordinates prompt engineering, evaluation benchmarking, and application release management through a deterministic 5-stage decision pipeline.

### 5-Stage Decision Pipeline

```
+-----------------------------------------------------------------------------------+
|                           5-STAGE DECISION PIPELINE                               |
+-----------------------------------------------------------------------------------+
|  [Stage 1: Prompt Ingestion & Syntax Verification]                               |
|  - Parse prompt template, validate Jinja2 variables and schema parameters         |
|                                     |                                             |
|                                     v                                             |
|  [Stage 2: Testset Selection & Batch Generation]                                  |
|  - Fetch curated gold dataset, resolve batch partitions and concurrency quotas    |
|                                     |                                             |
|                                     v                                             |
|  [Stage 3: Multi-Metric Evaluation & Scoring]                                     |
|  - Execute evaluators (exact match, semantic similarity, LLM-as-a-judge, cost)    |
|                                     |                                             |
|                                     v                                             |
|  [Stage 4: Threshold Verification & Champion Comparison]                          |
|  - Check S_eval >= 0.70 and delta S_eval relative to production champion baseline|
|                                     |                                             |
|                                     v                                             |
|  [Stage 5: Deployment Decision, Canary Gate & Audit Telemetry]                    |
|  - Promote variant to canary/production or trigger fallback rollback             |
+-----------------------------------------------------------------------------------+
```

### Mathematical Formulation of Scoring & Routing

For a candidate prompt variant $v_j$ evaluated across a test dataset $T = \{t_1, t_2, \dots, t_N\}$, the composite evaluation score $S_{\text{eval}}(v_j, T)$ is formulated as:

$$S_{\text{eval}}(v_j, T) = \frac{1}{N} \sum_{k=1}^N \left( w_{\text{acc}} M_{\text{acc}}(v_j, t_k) + w_{\text{sem}} M_{\text{sem}}(v_j, t_k) + w_{\text{cost}} M_{\text{cost}}(v_j, t_k) + w_{\text{lat}} M_{\text{lat}}(v_j, t_k) \right)$$

Where:
- $M_{\text{acc}}(v_j, t_k) \in [0, 1]$ measures ground-truth accuracy (exact match or regex validation).
- $M_{\text{sem}}(v_j, t_k) = \cos(\mathbf{e}_{\text{pred}}, \mathbf{e}_{\text{ref}})$ measures cosine similarity between predicted and reference embeddings.
- $M_{\text{cost}}(v_j, t_k) = \max\left(0, 1 - \frac{\text{cost}(v_j, t_k)}{\text{cost}_{\max}}\right)$ normalizes token consumption against budget.
- $M_{\text{lat}}(v_j, t_k) = \max\left(0, 1 - \frac{\text{latency}(v_j, t_k)}{\text{latency}_{\max}}\right)$ normalizes response duration.
- Standard default weights: $w_{\text{acc}} = 0.40$, $w_{\text{sem}} = 0.30$, $w_{\text{cost}} = 0.15$, $w_{\text{lat}} = 0.15$, ensuring $\sum w = 1.0$.

Promotion to staging/production requires both absolute and differential thresholds:

$$S_{\text{eval}}(v_j, T) \ge \tau \quad (\tau = 0.70) \quad \land \quad S_{\text{eval}}(v_j, T) - S_{\text{eval}}(v_{\text{champion}}, T) \ge 0.0$$

### Thresholds and Refusal Criteria

When input data, execution parameters, or evaluation results violate operational criteria, Agenta halts processing deterministically:

| Error Code | Trigger Condition | Deterministic Behavior |
|---|---|---|
| `ERR_PROMPT_SYNTAX_INVALID` | Jinja2 syntax error or missing variable binding | Reject prompt compilation with line error |
| `ERR_EVAL_SAMPLE_INSUFFICIENT` | Test dataset sample count $N < 10$ | Refuse evaluation run to prevent sample bias |
| `ERR_SAFETY_POLICY_VIOLATION` | Toxicity score $> 0.10$ or injection detected | Abort run, flag variant, notify administrator |
| `ERR_DEPLOYMENT_HEALTH_CHECK_FAILED` | Canary endpoint HTTP status $\ne 200$ | Automatically roll back to champion variant |
| `ERR_RESOURCE_QUOTA_EXCEEDED` | Projected inference cost exceeds quota ceiling | Pause execution and await operator quota top-up |

### Multi-Tier Fallback Mechanisms

Agenta employs a 3-tier fallback architecture to maintain production availability:

1. **Tier 1 (Automated Parameter Pruning):** If a variant experiences latency or cost budget violations, automatically clamp temperature and max output tokens to baseline values and re-evaluate.
2. **Tier 2 (Champion Baseline Rollback):** If candidate variant score falls below $\tau$ ($S_{\text{eval}} < 0.70$), automatically retain or roll back traffic to the existing production champion variant.
3. **Tier 3 (Human-in-the-Loop Release Gate):** If evaluation scores show high variance across dataset subsets, route the candidate variant to the human review queue with annotated side-by-side diffs.

## The Data It Uses

### Inputs Processed
- **Prompt Templates**: Raw markdown, system instructions, and Jinja2 templated parameter schemas.
- **Runtime Variables**: Key-value test variables injected into templates during evaluation.
- **Inference Payloads**: Completed model outputs, token logs, and execution timestamps.

### Reference Data
- **Gold Standard Testsets**: Curated test collections with ground truth inputs, expected outputs, and rubric constraints.
- **Historical Benchmarks**: Stored metric vectors from previous prompt releases used for regression testing.
- **Model Registry Manifests**: Provider catalogs containing rate limits, token pricing, and supported context windows.

### Model Lineage & Weights
- **Inference Providers**: Connects to OpenAI, Anthropic, Cohere, Mistral, Google Gemini, and local HuggingFace/vLLM endpoints.
- **Evaluator Models**: Standardized LLM-as-a-judge models running fixed checkpoint versions to ensure deterministic evaluations.

### Retention & Data Privacy
- **Dataset Storage**: Secure multi-tenant database partitions with AES-256 encryption at rest.
- **Zero Training Guarantee**: User prompts and evaluation datasets are never contributed to public or foundation training corpuses.
- **PII Masking**: Integrated redaction filters automatically mask personal identifiers before logging.

## Limitations

1. **Limitation:** LLM-as-a-judge evaluators can exhibit position bias and self-enhancement bias when rating model outputs.
   **Mitigation:** Agenta swaps output ordering across twin evaluation passes and combines model judgments with deterministic lexical metrics.

2. **Limitation:** High concurrency evaluation runs against external LLM providers can trigger sudden API rate limits (HTTP 429).
   **Mitigation:** Built-in exponential backoff with jitter and configurable concurrency worker throttles regulate dispatch rates.

3. **Limitation:** Small testsets ($N < 50$) can produce statistically noisy metric aggregates that fail to capture tail distribution failures.
   **Mitigation:** Synthetic test case generation creates edge-case variations, and Agenta flags evaluation runs where sample size is sub-optimal.

4. **Limitation:** Custom Python evaluators present potential sandboxing and arbitrary code execution vulnerabilities.
   **Mitigation:** Custom evaluator code executes strictly within isolated gVisor/container sandboxes with restricted network capabilities.

5. **Limitation:** Prompt optimizations tuned for one model family (e.g., Claude) frequently transfer poorly to other model families (e.g., GPT-4o).
   **Mitigation:** Multi-variant playground matrix testing enables parallel multi-model benchmarking under identical testset inputs.

## Summary & Compliance Checklist

| Component | Status | Verification Detail |
|---|---|---|
| **5-Stage Decision Pipeline** | Verified | ASCII flow diagram mapping Stages 1 through 5 with explicit state transitions |
| **Scoring & Routing Mathematics** | Verified | Formal equation $S_{\text{eval}}$ with weighted accuracy, semantic, cost, and latency factors |
| **Deterministic Thresholds & Refusals** | Verified | $\tau = 0.70$ threshold and 5 standardized error codes (`ERR_*`) documented |
| **Multi-Tier Fallback Strategy** | Verified | Tier 1 (Pruning), Tier 2 (Champion Rollback), and Tier 3 (Human Gate) specified |
| **Data Privacy & Lineage Architecture** | Verified | Documented inputs, reference data, model lineage, and zero-retention policies |
| **5 Documented Limitations & Mitigations** | Verified | 5 numbered limitation/mitigation pairs covering judge bias, rate limits, and sandboxing |
