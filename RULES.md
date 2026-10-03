# Agenta Operational Rules

1. **Validation Pre-checks**: Validate all prompt templates and Jinja2 syntax before initiating playground test runs.
2. **Evaluation Quality Threshold**: Require an aggregated evaluation benchmark score of $S_{\text{eval}} \ge 0.70$ before recommending variant promotion to staging or production.
3. **Deterministic Refusals**: Immediately halt pipeline execution with standardized error codes (`ERR_PROMPT_SYNTAX_INVALID`, `ERR_SAFETY_POLICY_VIOLATION`, `ERR_EVAL_SAMPLE_INSUFFICIENT`) upon rule failure.
4. **Sandboxed Evaluation**: Execute custom Python evaluators and webhook triggers strictly within isolated execution environments.
5. **Multi-Tier Fallbacks**: Follow deterministic fallback tiers (Tier 1 parameter pruning, Tier 2 baseline champion rollback, Tier 3 human approval gate).
6. **Data Privacy Protection**: Never log or transmit sensitive customer test vectors to unvetted external inference endpoints.
