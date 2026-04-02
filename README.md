# LLMOps and Production AI.

## How do you monitor LLM applications in production?

Monitor across 5 layers (product + model + retrieval/tools + system + security):

- Product/UX: task success rate, user drop-off, thumbs up/down, “regenerate” rate, time-to-resolution, escalation-to-human rate.
- Model output quality: hallucination/groundedness rate, refusal/over-refusal rate, formatting/schema validity, policy violations, language detection.
- RAG/tools: retrieval hit-rate, Recall@k proxy (does answer cite retrieved evidence?), citation precision, tool-call success rate, tool latency, tool error categories.
- System reliability: p50/p95 latency (end-to-end + per stage), timeouts, error rate, rate-limit events, GPU utilization, queue depth, cache hit rate.
- Security: prompt-injection attempts, data-exfil signals (long outputs, repeated sensitive patterns), anomalous usage, and per-user/tenant access denials.

> In interview terms: “I log every request with a trace-id and capture prompt + retrieved context ids + model version + tool calls + latency breakdown, then I build dashboards + alerts for quality and ops metrics, and run canary/A|B releases gated by offline eval + online guardrails.”

---

## What is LLM observability?

LLM observability = having enough end-to-end visibility to (1) debug failures, (2) measure quality and safety, and (3) control cost/latency for an LLM app.

It’s broader than infra monitoring because you need to observe:

- Inputs: prompt, system instructions, user message, and context length.
- Context: what documents/chunks were retrieved, what tools were called, and with what arguments.
- Outputs: final answer, structured fields, citations, policy decisions.
- Outcomes: user feedback / task success.

Plus standard telemetry: traces, metrics, logs.

---

## What are guardrails for LLMs, and how do you implement them?

Guardrails are controls that constrain and verify LLM behavior so the app is safe, compliant, and reliable.

Common guardrail categories + implementation:

1. Input guardrails
    - Prompt-injection detection, PII detection/redaction, jailbreak patterns.
    - AuthN/AuthZ checks (who is the user? what data/tools can they access?).
2. Retrieval/tool guardrails
    - Hard gate in the tool router: only allow tools the user is authorized to use; deny by default.
    - Enforce document-level permissions (RBAC) before returning retrieved chunks to the model.
3. Output guardrails
    - Policy checks: toxicity, self-harm, sexual content, hate/harassment, PII leakage.
    - Groundedness checks for RAG (answer must be supported by retrieved sources; otherwise abstain).
    - Format/schema validation (JSON / function-call schema); auto-repair or re-ask.
4. Runtime guardrails
    - Rate limits, max tokens, max tool calls, timeouts, circuit breakers, fallback models.

In production, these are implemented as deterministic middleware + model-based classifiers (e.g., a moderation model) + human review loops for flagged cases.

---

## How do you implement content filtering for AI outputs?

Typical pipeline:

1. Define the policy taxonomy (what is disallowed vs sensitive vs allowed).
2. Run an output classifier (moderation model or rules) on:
    - The raw generated text
    - Any citations/extracted content
    - (Optionally) tool outputs before they reach the user
3. Take an action based on severity:
    - Block + safe refusal
    - Redact sensitive spans (PII)
    - Rewrite with constraints (e.g., “no medical advice”, “no personal data”)
    - Escalate to human review
4. Log the decision + reason codes for auditing and to tune thresholds.

For PII, use detection + redaction both at ingestion (docs) and at generation time, and enforce tenant/user filters so restricted content never enters the context window.

---

## How do you estimate the cost of running an AI-powered feature in production?

Break cost into: model inference + retrieval/tools + infra + engineering overhead.

1. Inference cost (dominant for many apps)
    - Estimate tokens per request: input tokens (prompt + context + retrieved chunks) + output tokens.
    - Cost = (input_tokens/1K * input_price) + (output_tokens/1K * output_price).
    - Multiply by expected QPS / DAU and consider peak traffic.
2. GPU/self-hosted cost (if not per-token)
    - Capacity planning: throughput per GPU at target p95 latency (with batching).
    - GPUs needed = peak_requests_per_second / requests_per_second_per_GPU.
    - Monthly cost = GPU_hourly_rate * 24 * 30 * GPU_count (+ CPU, storage, networking).
3. RAG/tooling cost
    - Embedding cost: new docs/day * embedding_tokens/doc * embedding_price.
    - Vector DB: storage for embeddings + query cost (and replication).
    - Tool calls: external API costs (search, OCR, email, etc.).
4. Overheads + risk buffers
    - Retries, caching (reduces cost), safety checks, eval runs, and monitoring/log retention.

> Interview line: “I build a spreadsheet model with token histograms from logs, then compute p50/p95 tokens, cost per request, and monthly spend under expected traffic; I also run load tests to translate that into GPU capacity if self-hosting.”

---

## Explain the AI product lifecycle from ideation to production
1. Problem/ideation
    - Define the user + use case, success metrics, and constraints (latency, cost, privacy/security, UX).
    - Decide if AI is actually needed vs rules/search.
1. Data strategy
    - Identify data sources, labeling plan, and quality checks (schema integrity, missingness, leakage).
    - Set up dataset versioning and train/val/test splits (often time-based if drift matters).
1. Modeling / prototyping
    - Start with a baseline (could be prompt-only, small model, classical ML, or RAG).
    - Iterate with offline evaluation and error analysis; add guardrails and failure-mode tests (hallucination, prompt injection, unsafe outputs).
1. System design + production planning
    - Choose serving approach (managed endpoint vs self-hosted), scaling plan, and security model (IAM, network boundaries, secrets).
    - For RAG/enterprise: implement document-level permissions (RBAC) and ensure authorization filters are enforced before retrieval results are returned.[[1]](https://www.notion.so/31e88a4b53af8008b3b4d83c5c5b6359?pvs=21)
1. Deployment
    - CI/CD for model + prompt + retrieval configs.
    - Roll out with canary/shadow/A/B testing; monitor p50/p95 latency, error rates, and quality.
1. Monitoring + iteration
    - Monitor model quality (drift, calibration), system health, and security signals (tool-call logs, suspicious prompt patterns, DDoS/rate limits).[[1]](https://www.notion.so/31e88a4b53af8008b3b4d83c5c5b6359?pvs=21)
    - Build retraining / re-index triggers, regression eval suites, and incident-driven improvements.

---

## What is LLMOps, and how does it differ from traditional MLOps?

LLMOps = the set of practices to reliably build, deploy, monitor, secure, and iterate LLM-powered systems (apps that use prompts, tools, RAG, agents, and often multiple models). Compared to traditional MLOps, it typically adds or emphasizes:

- Prompt + context management: prompts, system policies, retrieval configs, tool schemas are “production artifacts” just like models.
- Non-determinism and qualitative eval: you need rubric-based evals, regression suites, multi-run variance checks, and robustness tests (paraphrases, long context, adversarial).[[2]](https://www.notion.so/AI-evaluation-frameworks-common-ways-teams-structure-evals-32d88a4b53af8099b73ceffe59828e61?pvs=21)
- Grounding + hallucination controls: strong focus on “supported-by-evidence” behavior, abstention, citations, and retrieval quality.
- Security: prompt injection, data exfiltration, and tool authorization are first-class problems (especially for agents).[[1]](https://www.notion.so/31e88a4b53af8008b3b4d83c5c5b6359?pvs=21)
- Cost/latency coupling: tokens, context length, and tool calls make cost and tail latency directly tied to product decisions.

Traditional MLOps is often more about: dataset → training → model artifact → deterministic-ish inference with clear scalar metrics (AUC/F1), plus standard drift monitoring.

---

## How do you serve LLMs in production?

A practical, interview-style answer:

1. Choose serving mode
- Managed endpoints (Bedrock/Vertex/etc.) for fast iteration and lower ops burden; self-hosting (often Kubernetes) for deeper control, custom latency work, and sensitive data/security requirements.[[1]](https://www.notion.so/31e88a4b53af8008b3b4d83c5c5b6359?pvs=21)
1. Build an inference service around the model
- API layer (authN/authZ, rate limiting), request validation, batching, and streaming responses when UX benefits.
- Caching where it makes sense (prompt/result cache; embedding cache for RAG).
1. Optimize performance for latency + throughput
- Profile and optimize the hotspots: attention kernels (FlashAttention-style), quantization, and graph fusion / CUDA graphs / TensorRT-style optimizations to reduce kernel-launch overhead and improve GPU utilization.[[3]](https://www.notion.so/31988a4b53af80578591e7f0ba0f1a27?pvs=21)[[1]](https://www.notion.so/31e88a4b53af8008b3b4d83c5c5b6359?pvs=21)
- Track p50/p95 latency and throughput after each change (production cares about tail latency).[[3]](https://www.notion.so/31988a4b53af80578591e7f0ba0f1a27?pvs=21)
1. Add reliability + safety guardrails
- Timeouts, retries (careful), circuit breakers, fallback models/routes, and structured logging.
- For RAG: enforce document-level permissions (RBAC) and filter-by-identity before retrieving/returning context to prevent cross-tenant leakage.[[1]](https://www.notion.so/31e88a4b53af8008b3b4d83c5c5b6359?pvs=21)
1. Observe and continuously improve
- Monitor: latency, GPU utilization, error rates, token usage, quality metrics, and security signals (tool-call logs, abnormal response lengths, suspicious prompts).[[1]](https://www.notion.so/31e88a4b53af8008b3b4d83c5c5b6359?pvs=21)

---

## What is model quantization?

Quantization = converting model weights and/or activations from higher-precision (FP32/FP16/BF16) to lower-precision (e.g., INT8, INT4, NF4) to reduce memory, bandwidth, and often speed up inference.

Key points to say in an interview:

- Why: smaller model footprint, faster matmuls, better throughput, lower latency—especially memory-bandwidth-bound workloads.
- Types:
    - Weight-only quantization (common for LLM inference)
    - Weight + activation quantization (more aggressive; can be harder to keep quality)
    - Post-training quantization vs quantization-aware training (QAT)
- Tradeoff: potential quality degradation; you validate on task-specific evals and watch for regressions.

---

## How do you optimize LLM inference costs in production?

Treat cost as (tokens + tool calls + infra) under a latency/quality SLA. The main levers:

1. Reduce tokens per request
    - Shorten system prompts, remove redundant instructions, and template prompts.
    - RAG: retrieve fewer but higher-quality chunks; use re-ranking; cap context length.
    - Summarize/compact conversation history (“memory”) instead of replaying full chat.
2. Reduce model $/token
    - Route requests: small/cheap model for easy intents; larger model only for hard cases.
    - Use fine-tuned smaller models for narrow tasks.
3. Reduce compute per token (self-hosted)
    - Quantization (INT8/INT4 / weight-only), KV-cache reuse, speculative decoding.
    - Efficient serving runtimes (vLLM / TRT-LLM style), dynamic batching, GPU utilization tuning.
4. Avoid waste
    - Caching: semantic cache for repeated questions; embedding cache for repeated docs.
    - Retry discipline: classify errors, don’t blindly retry long generations.
    - Put hard budgets: max tokens, max tool calls, timeouts.
5. Measure and iterate
    - Log token histograms (p50/p95), cost per request by endpoint/tenant, and correlate with task success.

---

## How do you implement A/B testing for LLM systems?

Run experiments at the “system” level (prompt + model + retrieval + guardrails), not just the base model.

1. Define variants
    - A = baseline prompt/model/retrieval config; B = changed component(s).
2. Randomization + isolation
    - Randomize by user/session (sticky assignment) to avoid cross-contamination.
    - Guardrails stay constant unless the guardrail itself is the treatment.
3. Metrics
    - Online: task success proxies (thumbs-up, completion, escalation), latency, cost/request, refusal rate, safety violations.
    - Offline: run a fixed eval suite + regression tests before exposing to users.
4. Rollout strategy
    - Start with shadow or canary (small %), then ramp.
    - Add automated stop conditions (spike in safety/latency/cost).
5. Analysis
    - Segment by intent, user cohort, and long-context vs short-context.

> Interview line: “I gate B behind offline eval + canary, then run a sticky user-level A/B and monitor both quality and safety/cost metrics; if B wins and stays within guardrails, I ramp to 100%.”

---

## What is CI/CD for AI applications, and how does it differ from traditional CI/CD?

CI/CD for AI covers more artifacts than just code:

- Traditional CI/CD: build/test/deploy code.
- AI CI/CD: build/test/deploy code + model weights + prompts + retrieval indexes + policies + eval suites.

What’s different in practice:

1. Tests are behavioral
    - Unit/integration tests still exist, but you also need offline evals (rubric/benchmarks), safety tests, and robustness suites.
2. Data + model lineage matters
    - Track dataset versions, labeling changes, training config, and reproducibility.
3. Deployment patterns
    - Canary/shadow deployments, model registry, rollback automation, and feature flags.
4. Observability is part of the pipeline
    - Automatic dashboards/alerts for quality + cost + latency regressions after release.

---

## How do you version and manage prompts in production?

Treat prompts like code:

1. Store prompts in a versioned repo/registry
    - Each prompt has an id, semantic version, owner, changelog, and links to eval results.
2. Use templates + variables
    - Keep system prompt stable; inject dynamic context (user, tool schemas, RAG snippets) via structured slots.
3. Release via feature flags
    - Roll out prompt changes with canary + A/B (sticky assignment).
4. Test prompts before shipping
    - Regression suite (golden prompts), safety tests (jailbreak, PII leakage), and schema/format validation.
5. Observability hooks
    - Log which prompt version produced each output (prompt_version, model_version, retrieval_version) for debugging.

---

## What is model versioning, and how do you handle model rollbacks?

Model versioning = assigning unique, immutable versions to model artifacts and tracking lineage (data, code, hyperparams, evals).

Practical approach:

1. Registry
    - Register every model with metadata: training data snapshot, config, dependencies, eval metrics, intended use.
2. Compatibility + safety
    - Validate against a fixed eval suite + safety gates before promotion to “staging” and “prod.”
3. Deployment with easy rollback
    - Serve behind a routing layer/feature flag: production points to a model alias (e.g., “prod”) that maps to a concrete version.
    - Rollback = repoint alias to the previous known-good version (fast), then post-mortem.
4. Rollback triggers
    - Automated if: safety violations increase, p95 latency spikes, cost/request increases, or task success drops.
5. Multi-artifact rollbacks
    - For LLM apps you often rollback a bundle: model + prompt + retrieval index + guardrail thresholds.

---

## How do you implement rate limiting and throttling for LLM APIs?

Goal: protect cost + latency + downstream dependencies while keeping UX predictable.

1. Decide quotas
    - Per user / per API key / per org(tenant) / per IP.
    - Separate budgets for: requests/min, tokens/min, concurrent requests, and tool calls/min.
2. Enforce at the edge + in the app
    - Edge (API gateway): cheap reject early (429) using token-bucket/leaky-bucket.
    - App layer: concurrency limits (semaphores/queues) and per-route budgets.
3. Prioritize and degrade gracefully
    - Tiered plans (free/pro/enterprise), priority queues.
    - Degrade: smaller model, shorter max_tokens, disable expensive tools, or return “try again later.”
4. Protect against abuse
    - Burst limits, anomaly detection, WAF rules.
    - Backpressure: shed load when GPU queue depth or p95 latency crosses thresholds.
5. Observability
    - Log limit hits by reason code (RPS vs TPM vs concurrency) and surface dashboards.

---

## How do you handle model updates and migrations without downtime?

Use “parallel run + gradual cutover” patterns:

1. Blue/green or canary deployments
    - Run old (A) and new (B) models side-by-side; shift traffic gradually.
2. Shadow testing
    - Send a copy of live traffic to B (no user impact) to measure quality/latency/cost before cutover.
3. Backward/forward compatibility
    - Keep request/response schemas stable; version APIs.
    - For embeddings/vector DB changes: maintain dual indexes during migration; backfill in background.
4. Stateful concerns
    - If prompts/tools change, treat it as a bundle migration (model + prompt + retrieval config).
    - Ensure cache keys include version (model_version/prompt_version) to avoid mixing.
5. Fast rollback
    - Route via an alias/feature flag so rollback is just a pointer flip.

---

## What is the role of feature flags in AI deployments?

Feature flags let you control exposure and rollback quickly for risky, behavior-changing systems.

Common uses:

- Rollout control: enable a new model/prompt/retrieval config for 1% → 10% → 50% → 100%.
- Experimentation: A/B tests with sticky assignment.
- Segmentation: enable features per tenant, plan tier, region, or internal users.
- Safety kill-switch: instantly disable tool use, reduce max_tokens, or revert to a safe baseline.
- Multi-artifact bundles: treat “LLM system version” as a flaggable package.

---

## How do you implement logging and tracing for LLM applications?

You want end-to-end traces that connect user request → retrieval/tools → model → post-processing.

1. Correlation ids
    - Assign trace_id + request_id; propagate through every service and tool call.
2. Structured logs (sanitized)
    - Log: model_version, prompt_version, retrieval_version, token counts, latency breakdowns, tool calls + status, moderation decisions.
    - Store raw text only when allowed; otherwise store hashes/redacted text.
3. Distributed tracing
    - Create spans for: gateway, retriever, reranker, LLM call, tool calls, guardrails, response assembly.
    - Export to an APM stack (OpenTelemetry-style) for flame graphs + p95 analysis.
4. Sampling
    - Sample full payload logs; always keep metrics.
    - Keep “full fidelity” for errors/safety events.
5. Privacy + retention
    - Encrypt logs, least-privilege access, retention limits, and audit trails.

---

## How do you handle PII and sensitive data in LLM inputs and outputs?

Layered approach (prevent it entering, prevent it being retrieved, prevent it leaking):

1. Data minimization
    - Don’t send sensitive fields unless needed; use opaque ids and fetch details server-side.
2. Detection + redaction
    - PII detection/redaction on input (and at ingestion for documents).
    - Re-run detection on output; redact or block if leakage is detected.
3. Access control
    - Enforce tenant/user RBAC before retrieval; never rely on the LLM to “respect permissions.”
4. Secret handling
    - Use a secrets manager; never embed secrets in prompts; rotate keys.
5. Policy + compliance
    - Clear retention policy for prompts/logs; encryption in transit/at rest; audit logging.
6. Model/provider choices
    - For high-sensitivity: self-host or use providers with strong data-handling guarantees; disable training on your data.

---

## Gateway pattern for LLM API management

A “gateway” is a dedicated service that sits between your clients (web/mobile/backend) and one or more LLM providers/models. It centralizes cross-cutting concerns so every app team doesn’t re-implement them.

Core responsibilities:

- Routing: choose provider/model per request (by tenant, use case, latency target, cost budget, region, etc.)
- AuthN/AuthZ: API keys, tenant isolation, per-user entitlements, tool permissions
- Policy + safety: input/output moderation, PII handling, jailbreak/prompt-injection heuristics
- Rate limiting + quotas: requests/min, tokens/min, concurrent streams, per-tenant budgets
- Observability: structured logs + traces, prompt/model/version tags, cost accounting
- Reliability: retries (careful), circuit breakers, timeouts, fallbacks, queueing/backpressure
- Standardization: one request/response schema across providers, unified streaming protocol
- Caching: semantic cache, prompt/result cache, KV/retrieval caches (where applicable)

A typical flow:

- Client → Gateway → (optional) RAG service / tool services → LLM provider(s) → Gateway post-processing → Client

---

## Streaming responses for real-time AI applications

Streaming is usually implemented as “send tokens/chunks as they’re generated” instead of waiting for the full completion. Two common transports:

- Server-Sent Events (SSE): simplest for unidirectional streams over HTTP
- WebSockets: when you need bi-directional, long-lived interactive sessions

Implementation checklist (what matters in production):

- Define a stream envelope: every chunk should have a type, e.g. `delta_text`, `tool_call`, `tool_result`, `final`, `error`, plus `request_id` for correlation.
- Backpressure and cancellation:
    - If the client disconnects, cancel the upstream LLM request (don’t keep generating tokens you can’t deliver).
    - Apply write timeouts and bounded buffers so slow clients don’t blow up memory.
- Heartbeats: periodically emit keep-alives to avoid idle timeouts at proxies/load balancers.
- Partial failure handling:
    - If a tool call fails mid-stream, decide whether to (a) end the stream with an error, (b) emit an “error chunk” and continue with a fallback, or (c) restart generation with a smaller plan.
- Streaming + tool calls:
    - Many teams stream “thinking-free text” until a tool call is required, then pause user-visible tokens, execute tool(s), then resume.
- Metrics for streaming:
    - Time-to-first-token (TTFT)
    - Tokens/sec (or chars/sec) during the stream
    - Stream interruption rate (client disconnects, proxy resets)

---

## Key SLAs + metrics for production AI systems (latency/throughput/availability)

Think in layers: user-perceived UX, system health, and model economics.

User-facing SLAs/SLOs (what users feel):

- Availability (successful responses / total), usually measured per endpoint + tenant
- End-to-end latency percentiles: p50/p90/p95/p99
- TTFT (for streaming UX)
- Quality SLOs: task success proxy (thumbs up/down), escalation rate, “regenerate” rate
- Safety/compliance: policy violation rate, PII leakage rate, refusal correctness (avoid over-refusal)

System reliability + throughput:

- Throughput: requests/sec, tokens/sec, concurrent streams
- Error rate by class: timeouts, provider 429s, provider 5xx, tool failures, validation failures
- Saturation: queue depth, concurrency, CPU/GPU utilization, memory pressure
- Tool/RAG metrics: tool-call success rate, tool latency p95, retrieval hit-rate, “answer grounded in citations” rate (if you do RAG)

Cost and efficiency:

- Tokens per request (p50/p95), context length distribution
- Cost per request / per successful task
- Cache hit rate
- Spend per tenant + anomaly detection (sudden spikes)

---

## Cloud vs on-device model deployment (high-level tradeoffs)

Cloud deployment tends to win on:

- Model size/capability: run larger models + faster iteration
- Centralized updates: ship improvements instantly
- Observability + control: easier monitoring, policy enforcement, analytics

On-device tends to win on:

- Privacy: sensitive data can stay local
- Offline/low-connectivity: works without network
- Latency consistency: avoids network variability (though compute may be slower)
- Cost at scale: reduce per-request inference spend (shift cost to device)

Key decision axes:

- Data sensitivity + compliance constraints
- Latency target and variability tolerance
- Model size vs device compute/memory budgets
- Update cadence (do you need rapid patching?)
- Product surface: is it okay if experience varies by device tier?

Common hybrid pattern:

- On-device for “fast, private, small” tasks (classification, short drafting, command intent)
- Cloud for “hard, long, tool-using” tasks (deep reasoning, large context, RAG, multi-step agents)

---

## Fallback strategies when primary model is unavailable or rate-limited

You want layered resilience, not just “try another model”.

Typical strategy stack:

- Timeouts + circuit breakers:
    - Set strict per-stage time budgets (gateway, retrieval, tool, LLM).
    - If a provider starts failing, open the circuit and stop sending traffic for a cool-off window.
- Retries (disciplined):
    - Retry only on clearly transient errors (some 5xx, network resets).
    - Avoid retrying long generations blindly; use capped retries and jittered backoff.
- Model/provider failover:
    - Primary → secondary provider (or same provider different region)
    - Big model → smaller model (degrade gracefully)
- Feature degradation:
    - Disable expensive tools, reduce max tokens, reduce retrieved context size, lower reasoning mode
- Queueing and admission control:
    - If saturated, shed load early with a friendly “try again” instead of timing out after 45s.
- Response shaping:
    - If you cannot complete, return an explicit partial + next action (“I couldn’t access tool X; here’s what I can answer without it.”)

Operationally, track:

- Fallback activation rate (too high = your primary is unhealthy or capacity is insufficient)
- Success rate after fallback (if low, your fallback isn’t actually viable)

---

## Structured output from LLMs reliably in production

Goal: make outputs machine-parseable with high success and safe failure modes.

A robust approach:

- Constrain generation:
    - Prefer provider features that enforce JSON/schema (when available).
    - Use “function calling / tool calling” style outputs where the model must produce arguments matching a schema.
- Validate deterministically:
    - Always run a JSON/schema validator (e.g., JSON Schema / Pydantic-like) after generation.
    - Reject extra fields if you need strictness; otherwise allow but ignore unknown fields.
- Repair loop (bounded):
    - If validation fails, do an automatic “repair” pass: provide the validation errors and ask the model to emit corrected JSON only.
    - Cap to 1–2 repair attempts; then fail gracefully.
- Separate “user text” from “machine output”:
    - Often: one structured object for your app + a separate natural-language message (or generate the user message from the structured object).
- Version your schemas:
    - Include `schema_version` in the output so you can evolve formats safely.
- Test with adversarial cases:
    - Long outputs, multilingual, special characters, empty fields, and tool-error scenarios.

If you want, I can answer each of these in “interview mode” (tight 30–60 second responses) vs “system design mode” (deeper architecture).

---

## Handling long contexts efficiently in production (compression + prefix caching)
- Context budgeting (first): set a hard token budget per request (input + output) and allocate it explicitly across system prompt, conversation history, retrieved docs, and tool outputs.
- Context compression patterns:
    - Summarize history into “state”: periodically replace older turns with a compact summary + a small set of pinned facts (user preferences, constraints, identifiers).
    - Query-focused compression: before retrieval, rewrite the user request into a minimal “search query + constraints” and only keep the parts of history that affect those constraints.
    - RAG compaction: deduplicate chunks, remove boilerplate, use a reranker to keep top-k high-signal passages, and strip irrelevant sections (headers, nav, footers).
- Prefix/KV caching (to reduce latency + cost):
    - Prefix caching: keep the invariant prefix identical across requests (system prompt + tools schema + formatting rules) so the provider/runtime can reuse cached computation.
    - KV cache reuse: in self-hosted serving (vLLM/TRT-LLM-style), reuse attention key/value states for shared prefixes across requests where feasible.
    - Practical tips: make your “static prefix” truly static (no timestamps, no random ids); version it and only change intentionally so cache hit rate stays high.
- Use a “two-pass” approach for very long inputs:
    - Pass 1: compress/summarize/extract into a compact intermediate representation.
    - Pass 2: run the main reasoning/generation over the compact form.

---

## Semantic routing + implementing it in a multi-model system

Semantic routing = automatically choosing which model/prompt/toolchain to use based on the meaning (intent/complexity/risk) of the request rather than static rules.

Implementation patterns:

- Embed-and-classify:
    - Compute an embedding for the user request (optionally plus a short context summary).
    - Compare to intent prototypes (nearest neighbors) or run a lightweight classifier.
    - Route to: cheap model vs strong model, tool-using agent vs no-tools, safe mode vs normal mode.
- LLM-as-router (bounded):
    - Use a small, cheap model to output a structured route decision (e.g., `{route: "code"|"faq"|"rag"|"sensitive", confidence: ...}`), then enforce with deterministic rules.
- Guardrails on routing:
    - High-risk categories (medical/legal/finance, sensitive data, self-harm, enterprise data) should force stricter policies and often a more reliable model + extra checks.
    - Add “abstain/uncertain” route → escalate to stronger model or ask a clarifying question (if your UX allows).
- Measure + iterate:
    - Track router confusion: misroutes, fallback rate after route selection, and per-route success/latency/cost.

---

## Managing secrets and API keys securely in LLM applications

Baseline rules:

- Never put secrets in prompts, logs, or client-side code.
- Use a secrets manager (AWS Secrets Manager / GCP Secret Manager / Vault) and short-lived credentials where possible.

Practical implementation:

- Server-side token brokerage:
    - Client authenticates to your backend; backend calls LLM provider with provider key.
    - If you must expose “keys” to clients (generally avoid), use ephemeral, scoped tokens with tight TTL and quotas.
- Least privilege + scoping:
    - Separate keys per environment (dev/staging/prod), per service, and sometimes per tenant.
    - Limit permissions (only required endpoints/models) and enforce per-tenant quotas at your gateway.
- Rotation + incident readiness:
    - Automate rotation; support hot reload of secrets without redeploying.
    - Audit usage: log key id (not the key), request ids, tenant, model, and cost attribution.
- Redaction + egress controls:
    - Redact known secret patterns in logs.
    - Lock down outbound traffic so only approved provider endpoints are reachable (prevents exfil to arbitrary hosts).

---

## Latency spikes during peak hours — stabilizing the LLM API

Diagnose first: spikes are usually saturation (queues), upstream rate limits, or noisy neighbors.

Stabilization levers (most impactful first):

- Concurrency control + backpressure:
    - Put explicit limits on in-flight LLM calls and tool calls; queue with bounded depth.
    - Shed load early (fast 429 / “try again”) rather than timing out after long waits.
- Adaptive routing:
    - During peak, route more traffic to cheaper/faster models, reduce max output tokens, reduce retrieved context, disable expensive tools.
- Caching:
    - Response cache for repeated queries; prefix/KV caching to reduce compute per request.
- Provider/rate-limit resilience:
    - Use multi-region or multi-provider failover; circuit break on elevated 429/5xx.
- Performance tuning:
    - For self-hosted: dynamic batching, better GPU utilization, quantization, and keeping model hot (avoid cold starts).
    - For managed APIs: ensure connection reuse, streaming (improves perceived latency), and request timeouts aligned with user UX.
- Operational SLOs:
    - Track p95/p99 end-to-end and per-stage (gateway, retrieval, provider). If p99 is bad, it’s often queueing—treat queue depth as a first-class metric.

---

## Costs too high — reducing costs without degrading quality

A proven “no-regret” order:

- Reduce wasted tokens:
    - Shorten system prompts; remove repeated instructions; cap history; summarize older turns.
    - RAG: retrieve fewer, better chunks (rerank), and dedupe.
- Route intelligently:
    - Use small/cheap model for easy intents; reserve large model for hard cases or low-confidence routes.
- Cut tool/retrieval spend:
    - Cache retrieval results; cache embeddings; avoid calling tools unless routing says it’s needed.
- Increase serving efficiency (self-hosted):
    - Quantization (weight-only INT8/INT4), speculative decoding, better batching, and higher GPU utilization.
- Improve first-pass success:
    - Better prompts + structured output validation to reduce retries/repair loops.
    - Track “regenerate” and internal retry rate—retries are a silent cost multiplier.

---

## Hitting LLM provider rate limits during peak hours — how to handle it  

- Prevent: enforce your own quotas before the provider
    - Token-bucket at the gateway per tenant/user/key for requests/min, tokens/min, and concurrent streams.
    - Admission control: cap in-flight LLM calls; bound queues; shed load early with fast 429 instead of slow timeouts.
- Adapt: degrade intelligently under pressure
    - Reduce max output tokens, lower context size/top-k retrieval, disable expensive tools.
    - Route more traffic to smaller/faster models; reserve best model for high-value/high-risk routes.
    - Cache: semantic cache for repeated queries; prefix/KV caching for shared prompts.
- Recover: use resilience patterns
    - Circuit breaker on elevated 429s; exponential backoff + jitter for small, safe retries.
    - Multi-provider or multi-region failover for the same “model class”.
- Measure:
    - Track rate-limit events by tenant/model, queue depth, retry rate, and “fallback activated” rate.

---

## Switching providers without downtime (dependency on one provider)  

- Put a provider-agnostic gateway in front:
    - Normalize request/response (incl. streaming) into your own schema.
    - Hide provider-specific features behind capability flags (function-calling, JSON schema, tool call formats).
- Run both providers in parallel during migration:
    - Shadow traffic: send copies to new provider (no user impact) to compare quality/latency/cost.
    - Canary: gradually shift 1% → 10% → 50% → 100% with automated rollback triggers.
- Version everything:
    - Prompt versions, tool schemas, safety policies, and response parsing logic must be versioned so you can flip traffic safely.
- Operational readiness:
    - Health checks per provider, circuit breakers, and a routing policy that can instantly disable a provider (kill switch).

---

## Scaling from 100 rps to 5000 rps (concurrency)  

- First identify the bottleneck: queueing vs compute vs downstream tools vs DB/retrieval. Then apply:
- Control concurrency explicitly:
    - Per-worker in-flight limits; global concurrency budgets; bounded queues with backpressure.
- Horizontal scale the stateless gateway and orchestrator:
    - Keep request handlers stateless; push state to caches/DB; use autoscaling on CPU + queue depth.
- For self-hosted inference:
    - Use high-throughput serving (dynamic batching, continuous batching), maximize GPU utilization, pin models in memory (avoid cold starts), consider quantization/speculative decoding.
    - Scale by adding replicas and sharding traffic; separate pools by model size.
- For managed providers:
    - Use multiple API keys/accounts where allowed, multiple regions, connection reuse, and a fallback provider.
- Separate critical paths:
    - Split “fast path” (cheap model/no tools) from “slow path” (tools/RAG/large model) so slow requests don’t starve the entire system.

---

## Peak traffic spike brings system down — handling peak traffic  

- Protect the system first:
    - Load shedding: prioritized queues + reject early for low-priority traffic.
    - Rate limiting per tenant; burst controls; WAF/DDOS protections if public-facing.
- Degrade gracefully:
    - Smaller model, shorter responses, fewer tools, fewer retrieved docs, or “summary-only” mode.
- Queue and schedule:
    - For non-interactive jobs, move to async: accept request, enqueue, process with workers, notify/return later.
- Capacity planning:
    - Pre-warm during known peak windows; autoscale on queue depth and saturation metrics; set SLO-based alerts.

---

## Eliminating single points of failure (provider outage took you down)  

- Multi-provider strategy:
    - At least one secondary provider (or self-hosted fallback) with a compatible “minimum viable” capability.
- Multi-region and redundancy:
    - If provider supports regions, route across regions; otherwise diversify providers.
- Circuit breakers + health-based routing:
    - Detect provider brownouts (latency + error spikes) and automatically drain traffic.
- Design for partial correctness:
    - If tools or best model unavailable, return a degraded answer with explicit limitations rather than failing the whole request.

---

## Multi-LLM pipeline breaks when one model fails — orchestration failure handling  

- Make every step resilient and typed:
    - Define step contracts (input/output schemas), validate outputs, and fail fast on schema violations.
- Retry policy per step:
    - Only retry idempotent steps; cap retries; classify errors (429/timeout vs bad output).
- Fallback per step:
    - If “model B” fails, route that step to another model or a simplified heuristic step.
- Compensation and checkpoints:
    - Persist intermediate results so you can resume rather than restart the whole chain.
- Time budgets:
    - Allocate time per step and enforce a hard end-to-end deadline; skip optional steps when near budget.

---

## Zero visibility into which step is failing — adding observability  

- End-to-end tracing:
    - Generate `trace_id`/`request_id` at ingress; propagate through every step and tool call.
    - Create spans for: gateway, router, retrieval, reranker, LLM call, tool calls, validators, post-processing.
- Structured logs (sanitized):
    - Log model/provider, prompt version, tokens in/out, latency per span, error codes, retry/fallback decisions.
- Metrics + dashboards:
    - p95/p99 latency per step, tool success rates, provider 429/5xx rates, queue depth, concurrency, cache hit rate.
- Debug sampling:
    - Sample full payloads for a small % (with redaction) and 100% for errors/safety events.

---

## Quantization caused accuracy drop — minimizing quantization loss  

- Choose the right quantization method:
    - Prefer weight-only quantization first; consider higher-bit (INT8) before INT4; use per-channel quantization when available.
- Calibrate correctly:
    - Use a representative calibration set (real prompts/context lengths) and measure task-level metrics, not just perplexity.
- Mixed precision / selective quantization:
    - Keep sensitive layers in higher precision (e.g., embeddings, output projection, or attention layers depending on findings).
- Use QAT or distillation when needed:
    - Quantization-aware training or distill from the FP16 teacher into the quantized student to recover quality.
- Guard with eval gates:
    - Maintain a regression eval suite (including long-context and tool-use cases) and block rollout if quality drops past threshold.

---

## Designing graceful degradation (one failing component shouldn’t take down platform)  

- Isolation:
    - Bulkheads: separate pools/queues for different tenants/features/models so one hot path can’t starve others.
    - Timeouts everywhere; bounded resources; no unbounded queues.
- Feature flags and kill switches:
    - Instantly disable expensive tools, disable streaming, reduce max tokens, or switch routing policies.
- Degraded modes:
    - “Answer without tools”, “short answer only”, “cached answer”, “retry later” depending on failure type.
- Fallback hierarchy:
    - Primary model → secondary provider → smaller model → template/rules-based minimal response.
- Clear user-facing behavior:
    - Return a consistent error/degraded response format so clients can handle it predictably.
