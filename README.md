# guardrails-

1. PROJECT OVERVIEW
Core Purpose
Guardrails++ is a policy-driven safety middleware gateway that intercepts LLM traffic at the API layer — screening both user prompts (input) and model responses (output) before either reaches its destination. The goal is to enforce organizational content policy without modifying the underlying model.

What It Detects and Prevents

Social engineering and financial fraud (phishing, CEO fraud, BEC, wire transfer scams)
Physical harm instructions (weapons, explosives, improvised devices)
Data exfiltration attempts (SSN, PII, credit card dumps, patient records)
Audit and detection evasion (bypassing SIEM, disabling logs)
Sensitive credential exposure (passwords, API keys, secrets)
General toxicity and harmful language (via ML classifiers)
Self-harm and disappearance planning (logged for review)
Problem It Solves
Enterprise LLM deployments have no native content enforcement layer. Teams cannot audit what prompts go in, what responses come out, or which policy rule caused a rejection. Guardrails++ adds a governance layer between the user and the model with full traceability — every decision is explainable, every trigger is named.

How It Works at a High Level
Every API call passes through a policy evaluation engine. The engine runs the text through a priority-ordered stack of deterministic rules (keywords, regex) and probabilistic ML classifiers. It returns one of three decisions — allow, transform, or block — with the exact rule IDs that fired.

2. TECHNICAL STACK
Layer	Technology
Web framework	FastAPI (Python)
ASGI server	Uvicorn
HTTP client	httpx (async)
Data modeling	Pydantic v2
Config / policy DSL	YAML (pyyaml)
ML safety classifiers	HuggingFace Inference API
Logging	Python logging + structured JSON
Storage	None — stateless, no database
ML Models Used

unitary/unbiased-toxic-roberta — A RoBERTa model fine-tuned on the Unbiased Toxicity dataset. Returns a toxicity probability score per input.
s-nlp/roberta_toxicity_classifier — A second RoBERTa toxicity classifier used as an ensemble cross-check. Provides independent scoring to reduce both false positives and false negatives.
Important clarification on the "NLI" framing: These models are transformer-based text classifiers, not NLI (Natural Language Inference) models in the strict sense. True NLI models classify entailment/contradiction between a premise and hypothesis. The models here perform single-sequence toxicity classification — scoring how likely a given text is harmful. The architecture is RoBERTa-based, which is a bidirectional transformer encoder.

Hosting / Deployment
Models are hosted and served by HuggingFace's Inference API (router.huggingface.co). The application calls them over HTTPS at inference time — no local model loading, no GPU required. Authentication uses a bearer token (HF_API_TOKEN env var). A DemoSafetyClient fallback exists for offline testing.

3. GUARDRAILS IMPLEMENTATION
Three Policy Types

Type	How It Works
keyword_match	Lowercased substring scan across a keyword list
regex_replace	Compiled regex pattern match with optional substitution
external_safety_api	Async HTTP call to HuggingFace; compares risk score against configurable threshold
Content Categories Covered

Category	Policy ID	Action
Credential leakage	block_sensitive_keywords	Block
Email PII in outputs	mask_email_addresses	Rewrite (redact)
Confidential phrases	log_potential_confidential	Log only
Weapons / explosives	block_weapons_and_bombs	Block
Social engineering	safe_social_engineering	Safe completion
Physical harm techniques	safe_physical_harm_techniques	Safe completion
Data exfiltration	safe_data_exfiltration, safe_db_extraction	Safe completion
Audit evasion	safe_audit_evasion	Safe completion
Self-harm themes	log_self_harm_fiction	Log only
Disappearance planning	log_disappearance_planning	Log only
Toxicity (ML)	toxicity_unbiased_unitary, toxicity_snlp	Block
Actions on Violation

block → Return HTTP 400 with rules_triggered[] list. Prompt never reaches the model.
safe_complete → Return HTTP 200 with a curated deflection response specific to the category (social engineering, physical harm, data exfiltration, etc.). The user receives helpful context without the harmful content being generated.
rewrite → Mutate the text in-place (e.g., redact email addresses) and continue the pipeline with the cleaned version.
log_only → Flag for audit trail, allow the request to proceed.
Bidirectional Monitoring
Yes — the router runs two evaluation passes: one on the incoming user prompt (direction="input") and a second on the LLM response text (direction="output"). Output policies can catch model responses that slip through input filters.

4. ARCHITECTURE & WORKFLOW
Data Flow

User Request
    │
    ▼
CorrelationIDMiddleware          ← assigns X-Correlation-ID UUID
    │
    ▼
POST /v1/chat/completions
    │
    ▼
evaluate_policies_for_request()  ← INPUT pass
    ├── keyword_match policies   (synchronous, sequential)
    ├── regex_replace policies   (synchronous, sequential)
    └── external_safety_api      (async parallel via asyncio.gather)
    │
    ▼
Decision: allow / transform / block
    │
    ├── block    → 400 + rules_triggered[]
    ├── safe_complete → 200 + category-specific deflection text
    └── allow/rewrite → forward to LLM proxy
                            │
                            ▼
                    evaluate_policies_for_request()  ← OUTPUT pass
                            │
                            ▼
                    Final response returned to user
Policy Engine Priority
Deterministic rules (keyword, regex) run first synchronously. External ML classifiers run last in parallel via asyncio.gather() — only if no prior rule has already issued a block. This minimizes external API calls when a cheap keyword match would have been sufficient.

Decision Object
Every evaluation returns a typed Decision dataclass:

Decision(
    decision: "allow" | "transform" | "block",
    action: "pass_through" | "safe_completion" | "rewrite" | "reject",
    rules_triggered: List[str],
    rewritten_text: Optional[str],
    metadata: Dict,        # includes external ML scores
    status_code: int       # 200 or 400
)
Logging and Auditing

CorrelationIDMiddleware generates a UUID per request, injects it into request.state, and returns it as X-Correlation-ID response header
log_decision() writes a structured JSON log line per decision including correlation ID, method, path, status code, and duration in milliseconds
External ML risk scores and labels are captured in decision.metadata["external_safety"] keyed by policy ID
5. KEY FEATURES
API Endpoints

Method	Endpoint	Purpose
POST	/v1/chat/completions	Main gateway — evaluate and respond
GET	/v1/debug/policies	List all loaded policies with metadata
POST	/v1/debug/policies/reload	Hot-reload YAML without server restart
GET	/v1/debug/eval?text=...&direction=input	Test evaluation against a raw string
Configuration

All rules are defined in policies/base_policies.yaml — no code changes required to add or modify rules
Per-policy threshold (ML classifiers), severity, action, and applies_to are all configurable in YAML
Hot-reload via /v1/debug/policies/reload clears the in-memory cache and re-parses the YAML immediately
Real-Time Processing
Fully synchronous request/response. No batch processing or queue. External classifier calls are async and parallelized, with a 20-second timeout per HuggingFace request.

No Dashboard
There is no UI. Observability is via structured JSON logs intended for ingestion into any log aggregation system (Datadog, Splunk, ELK, CloudWatch, etc.).

6. IMPLEMENTATION HIGHLIGHTS
Deterministic + Probabilistic Hybrid
The most architecturally notable decision is layering deterministic rules (guaranteed, fast, auditable) with probabilistic ML scoring (catches novel harmful content that doesn't match any keyword). Deterministic rules always run first — if they block, the ML API call is skipped entirely, saving latency and cost.

Safe Completion vs. Blocking
Rather than hard-blocking everything suspicious, the system supports a safe_complete path that returns a curated, category-specific response. For example, a social engineering prompt gets a response that refuses the scam template but offers defender-perspective alternatives (DMARC controls, dual-approval workflows). This reduces friction for legitimate security training use cases while still preventing misuse.

Ensemble Toxicity Scoring
Two independent RoBERTa toxicity classifiers run in parallel on every request that clears deterministic checks. Each has its own threshold. Either can independently trigger a block. This ensemble approach reduces reliance on any single model's blind spots.

Policy Isolation via applies_to
Each policy explicitly declares whether it applies to input, output, or both. This prevents output-only rules (like email redaction) from incorrectly blocking user prompts, and prevents input-only rules from re-triggering on the LLM's own generated text.

Fail-Open on ML Errors
If the HuggingFace API is unreachable or returns an error, the safety client returns risk_score=0.0 and label="error". The engine treats this as below threshold and allows the request through rather than causing a service outage. This is an explicit operational tradeoff — availability over maximum safety — appropriate for middleware that should degrade gracefully.

Latency Consideration
Deterministic rules add ~1ms. External ML calls add 300–2000ms depending on HuggingFace model warm-up state. The two classifiers run concurrently, so the latency cost is the slower of the two, not the sum. This is the primary performance bottleneck in the current design.
