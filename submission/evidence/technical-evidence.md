# Day 13 Technical Evidence

## Pytest

```text
22 passed, 2 warnings
```

Command:

```bash
python -m pytest -q
```

## Log Validation

```text
--- Lab Verification Results ---
Total log records analyzed: 21
Records with missing required fields: 0
Records with missing enrichment (context): 0
Unique correlation IDs found: 10
Potential PII leaks detected: 0

--- Grading Scorecard (Estimates) ---
+ [PASSED] Basic JSON schema
+ [PASSED] Correlation ID propagation
+ [PASSED] Log enrichment
+ [PASSED] PII scrubbing

Estimated Score: 100/100
```

Command:

```bash
python scripts/validate_logs.py
```

## Dashboard Validation

```text
HỢP LỆ: 6/6 panel có trong dashboard contract.
```

Command:

```bash
python scripts/validate_dashboard.py
```

## Correlation ID Evidence

Example request/response pair from `data/logs.jsonl`:

```json
{"event":"request_received","correlation_id":"req-9dc84710","feature":"qa","model":"claude-sonnet-4-5","env":"dev","user_id_hash":"2055254ee30a","session_id":"s01"}
{"event":"response_sent","correlation_id":"req-9dc84710","latency_ms":150,"tokens_in":36,"tokens_out":179,"cost_usd":0.002793,"quality_score":0.9}
```

## PII Redaction Evidence

Raw sample inputs contained email, Vietnamese phone number, and credit card-like value. Logs contain redacted values:

```text
[REDACTED_EMAIL]
[REDACTED_PHONE_VN]
[REDACTED_CREDIT_CARD]
```

No raw PII was detected by `scripts/validate_logs.py`.

## Prompt Trace Evidence

The code links prompt metadata into Langfuse trace/generation when `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY` are configured:

```text
prompt_name
prompt_label
prompt_version
prompt_source
```

In the current local run, tracing is disabled because no Langfuse keys are configured, so the agent uses the visible local prompt fallback.
