# EXPLAINABILITY.md

This document explains the internal mechanisms, data lineage, operational boundaries, and governance framework of **Agenta** (`agenta`) in accordance with the **OpenGAP v0.1.0** specification for the **HiDevs GitAgent Passport** clearance pipeline.

> **Agent Name:** Agenta (`agenta`)  
> **Specification:** OpenGAP v0.1.0  
> **Category / Domain:** LLMOps / Prompt Engineering & LLM Evaluation  
> **Compliance Standard:** OpenGAP Checkpoint 2 (Explainability & Decision Governance), OWASP LLM Top 10, MITRE ATLAS  

---

## How the Agent Decides

Agenta coordinates prompt engineering, evaluation benchmarking, and application release management through a deterministic 5-stage decision pipeline.

### 1. Decision Architecture

The runtime intake, state classification, evaluation, and execution tracking operate across a deterministic, five-stage pipeline:

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

### 2. Decision Logic & Routing Formulations

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

### 3. Thresholding & Refusal Decision Criteria

Agenta enforces strict operational boundaries and deterministic refusal thresholds:
- **Refusal on ERR_PROMPT_SYNTAX_INVALID**: Jinja2 syntax error or missing variable binding halts execution with code `ERR_PROMPT_SYNTAX_INVALID`.
- **Refusal on ERR_EVAL_SAMPLE_INSUFFICIENT**: Test dataset sample count $N < 10$ halts execution with code `ERR_EVAL_SAMPLE_INSUFFICIENT`.
- **Refusal on ERR_SAFETY_POLICY_VIOLATION**: Toxicity score $> 0.10$ or injection detected halts execution with code `ERR_SAFETY_POLICY_VIOLATION`.
- **Refusal on ERR_DEPLOYMENT_HEALTH_CHECK_FAILED**: Canary endpoint HTTP status $\ne 200$ halts execution with code `ERR_DEPLOYMENT_HEALTH_CHECK_FAILED`.
- **Refusal on ERR_RESOURCE_QUOTA_EXCEEDED**: Projected inference cost exceeds quota ceiling halts execution with code `ERR_RESOURCE_QUOTA_EXCEEDED`.

### 4. Fallback Decision Mechanism

Continuous operational stability is maintained through layered fault recovery:
- **Tier 1 (Automated Parameter Pruning):** If a variant experiences latency or cost budget violations, automatically clamp temperature and max output tokens to baseline values and reevaluate.
- **Tier 2 (Champion Baseline Rollback):** If candidate variant score falls below $\tau$ ($S_{\text{eval}} < 0.70$), automatically retain or roll back traffic to the existing production champion variant.
- **Model Fallback Cascade**: High-level reasoning and synthesis default to `gemini-2.0-flash` with automatic failover to `gpt-4o` and `claude-3-5-sonnet`.

### 5. Human-in-the-Loop Governance

Human operators retain sovereign authority over the multi-agent execution lifecycle:
- **Tier 3 (HumanintheLoop Release Gate):** If evaluation scores show high variance across dataset subsets, route the candidate variant to the human review queue with annotated sidebyside diffs.
- **Session Telemetry Auditing**: Operators inspect execution logs, routing traces, and token usage to maintain oversight.

---

## The Data It Uses

Agenta operates under strict principles of data minimization, environment isolation, and privacy protection.

### 1. Ingested Input Data

The framework processes only operational data necessary to perform its functions:
- **Prompt Templates**: Raw markdown, system instructions, and Jinja2 templated parameter schemas.
- **Runtime Variables**: Key-value test variables injected into templates during evaluation.
- **Inference Payloads**: Completed model outputs, token logs, and execution timestamps.

### 2. Configuration & Reference Data

- **Gold Standard Testsets**: Curated test collections with ground truth inputs, expected outputs, and rubric constraints.
- **Historical Benchmarks**: Stored metric vectors from previous prompt releases used for regression testing.
- **Model Registry Manifests**: Provider catalogs containing rate limits, token pricing, and supported context windows.

### 3. Base Model & Inference Lineage

- **Inference Providers**: Connects to OpenAI, Anthropic, Cohere, Mistral, Google Gemini, and local HuggingFace/vLLM endpoints.
- **Evaluator Models**: Standardized LLM-as-a-judge models running fixed checkpoint versions to ensure deterministic evaluations.

### 4. Data Privacy, Storage, and Retention

- **OWASP LLM & MITRE ATLAS Hardened**: Defended against indirect prompt injection, credential leakage, and unauthorized external API dispatch.
- **Local Environment Isolation**: Agent execution workspaces, intermediate scratchpads, and vector stores reside strictly within designated local project directories.
- **Automated Secret Scrubbing**: API keys, database credentials, and personal credentials are automatically redacted prior to embedding or logging.
- **Zero Commercial Monetization**: Prompts, intermediate reasoning trajectories, and task deliverables are never commercialized or shared with third parties.

---

## Limitations

Understanding the operational boundaries and technical constraints of Agenta is essential for effective deployment.

### 1. LLM-as-a-judge evaluators can exhibit position bias
- **Limitation**: LLM-as-a-judge evaluators can exhibit position bias and self-enhancement bias when rating model outputs.
- **Mitigation**: Agenta swaps output ordering across twin evaluation passes and combines model judgments with deterministic lexical metrics.

### 2. High concurrency evaluation runs against external
- **Limitation**: High concurrency evaluation runs against external LLM providers can trigger sudden API rate limits (HTTP 429).
- **Mitigation**: Built-in exponential backoff with jitter and configurable concurrency worker throttles regulate dispatch rates.

### 3. Small testsets ($N < 50$) can
- **Limitation**: Small testsets ($N < 50$) can produce statistically noisy metric aggregates that fail to capture tail distribution failures.
- **Mitigation**: Synthetic test case generation creates edge-case variations, and Agenta flags evaluation runs where sample size is sub-optimal.

### 4. Custom Python evaluators present potential sandboxing
- **Limitation**: Custom Python evaluators present potential sandboxing and arbitrary code execution vulnerabilities.
- **Mitigation**: Custom evaluator code executes strictly within isolated gVisor/container sandboxes with restricted network capabilities.

### 5. Prompt optimizations tuned for one model
- **Limitation**: Prompt optimizations tuned for one model family (e.g., Claude) frequently transfer poorly to other model families (e.g., GPT-4o).
- **Mitigation**: Multi-variant playground matrix testing enables parallel multi-model benchmarking under identical testset inputs.

---

## Summary & Compliance Checklist

| Checkpoint 2 Requirement | Corresponding Section | Status |
| :--- | :--- | :---: |
| **How the agent decides** | [How the Agent Decides](#how-the-agent-decides) | **Covered** |
| - Decision architecture & 5-stage pipeline | Section 1 | Verified |
| - Decision logic & routing formulations | Section 2 | Verified |
| - Thresholding & refusal decision criteria | Section 3 | Verified |
| - Fallback decision mechanism | Section 4 | Verified |
| - Human-in-the-loop governance & oversight | Section 5 | Verified |
| **The data it uses** | [The Data It Uses](#the-data-it-uses) | **Covered** |
| - Ingested input data & query streams | Section 1 | Verified |
| - Configuration & reference schemas | Section 2 | Verified |
| - Base model lineage & deterministic engines | Section 3 | Verified |
| - Data privacy, retention lifecycle & MITRE/OWASP | Section 4 | Verified |
| **Its limitations** | [Limitations](#limitations) | **Covered** |
| - LLM-as-a-judge evaluators can exhibit position bias | Section 1 | Verified |
| - High concurrency evaluation runs against external | Section 2 | Verified |
| - Small testsets ($N < 50$) can | Section 3 | Verified |
| - Custom Python evaluators present potential sandboxing | Section 4 | Verified |
| - Prompt optimizations tuned for one model | Section 5 | Verified |
