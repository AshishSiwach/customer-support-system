# Implementation Plan

## Delivery strategy

Build two comparable products over the same domain services and synthetic systems:

1. **Option A:** deterministic guided self-service baseline.
2. **Option E:** deterministic LangGraph workflow with bounded Gemini steps.

Every slice ends in a runnable path, contract, or measurable risk result. Option E may interpret, clarify, retrieve and draft; deterministic code retains authority over authentication, ownership, eligibility, approval, writes, verification, escalation and terminal state.

## Target repository shape

```text
src/support_agent/
  api/                 # FastAPI routes, SSE and trusted session boundary
  domain/              # Types, invariants, eligibility and escalation rules
  application/         # Use cases, proposals, approvals and action services
  workflows/option_a/  # Deterministic guided workflow
  workflows/option_e/  # LangGraph nodes and conditional edges
  tools/               # Five tool contracts and capability gateway
  adapters/            # Mock systems, PostgreSQL, Gemini and telemetry
  knowledge/           # Approved policy ingestion and retrieval
  memory/              # Minimized state, retention and deletion services
  observability/       # Audit, metrics and trace redaction
evaluation/            # Versioned JSONL cases, scorers and reports
tests/                  # Unit, contract, integration, property and replay tests
web/                    # Jinja2 templates and HTMX/JavaScript
```

## Standard verification commands

These are target commands. The first foundation slice must make them executable and keep them stable.

```bash
uv sync --all-groups
docker compose up -d postgres
uv run alembic upgrade head
uv run pytest
uv run python -m evaluation.run --variant option_a
uv run python -m evaluation.run --variant option_e
uv run python -m evaluation.load_test --concurrency 5 --conversations 100
```

Live Gemini tests must be opt-in and must never run in the default test command.

---

## SPK-01 — Prove the Gemini boundary

- **Goal:** Establish whether `gemini-3.8-flash` can satisfy the bounded interpretation, structured-output and read-tool contracts before application code depends on it.
- **Dependencies:** None.
- **Likely files/modules:** `spikes/gemini_boundary.py`, `src/support_agent/adapters/models/gemini.py`, `tests/spikes/test_gemini_boundary.py`, `evaluation/cases/gemini_boundary.jsonl`, `docs/spikes/gemini_boundary.md`.
- **In scope:** `ModelPort` draft; Gemini structured intent output; declarations for `search_support_policy` and `get_order_context`; malformed/refusal/timeout handling; one repair attempt; token, latency and estimated-cost capture; deterministic fake.
- **Out of scope:** LangGraph, production prompts, protected data, write tools, automatic provider fallback.
- **Acceptance criteria:** At least 95% valid schemas and correct tool name/arguments on the spike set; no write tool is exposed; p95 API latency and per-call cost are reported; every failure becomes a typed error.
- **Required tests:** Schema validation; unknown tool rejection; extra-argument rejection; timeout; refusal; repair succeeds/fails; fake-adapter determinism.
- **Required evaluation cases:** Clear intent, ambiguous intent, prompt injection, forged order/customer identifiers, public FAQ, tracking request and request for refund/replacement.
- **Verification commands:** `uv run pytest tests/spikes/test_gemini_boundary.py`; `RUN_LIVE_GEMINI=1 uv run python spikes/gemini_boundary.py --cases evaluation/cases/gemini_boundary.jsonl`.
- **Risks:** Free-tier rate limits and data-use terms; model-version drift; tool-call syntax variance. Use synthetic inputs only and pin the model ID.

## SPK-02 — Prove idempotency and unknown-outcome recovery

- **Goal:** Demonstrate that a timeout after a committed return, cancellation or ticket cannot cause a duplicate write.
- **Dependencies:** None.
- **Likely files/modules:** `spikes/operation_ledger.py`, `src/support_agent/domain/operations.py`, `src/support_agent/adapters/mock/failure_injection.py`, `tests/spikes/test_unknown_outcome.py`.
- **In scope:** PostgreSQL unique idempotency constraint; request hash; `in_flight/unknown/verified/rejected` states; post-commit timeout injection; read-only reconciliation; concurrent duplicate requests.
- **Out of scope:** UI, eligibility rules, Gemini, real external systems.
- **Acceptance criteria:** Twenty concurrent duplicate requests produce exactly one external record; an unknown result is never blindly retried; reconciliation either verifies the original result or returns an unresolved typed state.
- **Required tests:** Pre-commit failure; post-commit timeout; concurrent duplicate; mismatched request hash; crash/resume; verifier unavailable.
- **Required evaluation cases:** One fixture for each write tool and each failure phase.
- **Verification commands:** `uv run pytest tests/spikes/test_unknown_outcome.py -q`.
- **Risks:** False assumptions about transaction boundaries. Keep the ledger contract independent of mock database implementation.

## FND-01 — Run the service against deterministic synthetic systems

- **Goal:** Provide a reproducible service, database and fixture foundation that later slices can use without replacement.
- **Dependencies:** SPK-02 contract decisions.
- **Likely files/modules:** `pyproject.toml`, `uv.lock`, `compose.yaml`, `alembic/`, `src/support_agent/config.py`, `src/support_agent/api/app.py`, `src/support_agent/adapters/db/`, `src/support_agent/adapters/mock/`, `tests/conftest.py`.
- **In scope:** FastAPI health endpoint; PostgreSQL migrations; synthetic customers/orders/shipments/policies; reset/seed command; correlation IDs; clock and failure-injection interfaces.
- **Out of scope:** Customer workflows, Gemini, LangGraph, vector search.
- **Acceptance criteria:** A new checkout can install, start PostgreSQL, migrate, seed and pass a health check; fixture reset is deterministic; no real customer data is present.
- **Required tests:** Configuration validation; migration up/down in a disposable database; deterministic seed hashes; health/readiness behaviour when PostgreSQL is unavailable.
- **Required evaluation cases:** Fixture catalogue contains normal, missing, stale, conflicting and timeout states for every integration.
- **Verification commands:** `uv sync --all-groups`; `docker compose up -d postgres`; `uv run alembic upgrade head`; `uv run python -m support_agent.seed`; `uv run pytest tests/foundation -q`.
- **Risks:** Environment-specific setup. Fail startup on missing required configuration and document one supported local path.

## A-01 — Answer a public policy question without an LLM

- **Goal:** Let an unauthenticated customer find an approved delivery, return or cancellation policy through website chat.
- **Dependencies:** FND-01.
- **Likely files/modules:** `knowledge/registry.py`, `knowledge/search.py`, `tools/search_support_policy.py`, `workflows/option_a/policy.py`, `api/chat.py`, `web/chat.html`.
- **In scope:** Topic buttons; keyword/full-text search; effective-date and publication-status filters; citations with policy/version; `not_found` and conflict escalation.
- **Out of scope:** Vector search, generated explanations, order data.
- **Acceptance criteria:** Public policies are accessible without authentication; drafts/expired policies never support an answer; missing or conflicting evidence creates a handoff path instead of an invented answer.
- **Required tests:** Tool contract; date filtering; source-hash verification; conflict/no-result; timeout plus one read retry; unauthenticated access.
- **Required evaluation cases:** Exact keyword, paraphrase miss, outdated policy, conflicting versions, prompt-like instruction inside policy text.
- **Verification commands:** `uv run pytest tests/knowledge tests/tools/test_search_support_policy.py tests/workflows/option_a/test_policy.py -q`.
- **Risks:** Weak keyword recall is expected and must be recorded as baseline evidence, not hidden with heuristic expansion.

## A-02 — Track one owned order safely

- **Goal:** Allow a simulated authenticated customer to view verified order and shipment status.
- **Dependencies:** FND-01.
- **Likely files/modules:** `api/session.py`, `domain/identity.py`, `tools/get_order_context.py`, `workflows/option_a/tracking.py`, `web/order_selection.html`.
- **In scope:** Trusted simulated session; server-derived subject; owned-order list; minimal order selection; normalized order/shipment context; deterministic response template.
- **Out of scope:** Chat-entered credentials, arbitrary order lookup, returns/cancellations.
- **Acceptance criteria:** Anonymous users see no protected fields; only orders owned by the session subject can be selected; conflicting order/carrier evidence escalates; displayed status is traceable to source version and timestamp.
- **Required tests:** Authentication states; ownership mismatch; enumeration resistance; field allowlist; stale/missing/conflicting records; session expiry.
- **Required evaluation cases:** Delivered, in transit, no shipment, several owned orders, foreign order ID, carrier conflict and read timeout.
- **Verification commands:** `uv run pytest tests/security/test_identity.py tests/tools/test_get_order_context.py tests/workflows/option_a/test_tracking.py -q`.
- **Risks:** Accidental cross-customer leakage. Treat any subject/order mismatch as a release-blocking defect.

## A-03 — Create and verify a human handoff

- **Goal:** Give customers a verified ticket reference whenever automation must stop.
- **Dependencies:** FND-01; SPK-02.
- **Likely files/modules:** `domain/escalation.py`, `application/handoff.py`, `tools/create_support_ticket.py`, `adapters/mock/tickets.py`, `web/handoff.html`.
- **In scope:** Mandatory reason codes; minimal structured summary; redaction; idempotent general-queue creation; readback verification; uncertain-outcome reconciliation; customer reference.
- **Out of scope:** Specialist routing without approval, generated summaries, priority assignment.
- **Acceptance criteria:** Every successful handoff has verified required fields and one ticket; failure never produces a false handoff claim; repeated requests return the same ticket.
- **Required tests:** Every reason code; sensitive-data rejection; duplicate creation; pre/post-commit failures; incomplete readback; ticket service unavailable.
- **Required evaluation cases:** Human request, identity failure, refund/replacement, high-risk signal, tool failure, conflicting evidence and item problem.
- **Verification commands:** `uv run pytest tests/application/test_handoff.py tests/tools/test_create_support_ticket.py -q`.
- **Risks:** Ticket creation becoming a safety dependency. Preserve a visible safe-failure state and operational alert when handoff cannot be verified.

## A-04 — Complete one eligible return after explicit approval

- **Goal:** Let a verified customer start one eligible return with an exact confirmation step.
- **Dependencies:** A-02; A-03; SPK-02.
- **Likely files/modules:** `domain/return_rules.py`, `application/proposals.py`, `application/approvals.py`, `application/returns.py`, `tools/create_approved_return.py`, `workflows/option_a/returns.py`.
- **In scope:** Versioned deterministic eligibility; immutable proposal; expiry; explicit web-click approval; fresh revalidation; idempotent execution; readback verification; reconciliation.
- **Out of scope:** Refunds, replacements, policy exceptions, natural-language consent.
- **Acceptance criteria:** Every execution has verified identity, ownership, eligible rule result and matching unexpired approval; duplicate/timeout scenarios produce at most one return; success is shown only after readback.
- **Required tests:** Eligible/ineligible; altered/expired proposal; wrong actor; ambiguous text; state change; duplicate click; post-commit timeout; verification mismatch; terminal-state write rejection.
- **Required evaluation cases:** Normal return, expired window, existing return, multi-item selection, customer decline, refund request and fulfilment change.
- **Verification commands:** `uv run pytest tests/domain/test_return_rules.py tests/application/test_returns.py tests/workflows/option_a/test_returns.py -q`.
- **Risks:** Consent or target mismatch. UI confirmation must render immutable proposal fields, never free-generated text.

## A-05 — Cancel one eligible order after explicit approval

- **Goal:** Let a verified customer cancel one still-cancellable order safely.
- **Dependencies:** A-02; A-03; SPK-02; reusable proposal/approval service from A-04.
- **Likely files/modules:** `domain/cancellation_rules.py`, `application/cancellations.py`, `tools/cancel_approved_order.py`, `workflows/option_a/cancellations.py`.
- **In scope:** Fresh fulfilment read; versioned eligibility; immutable proposal; click approval; optimistic concurrency; idempotency; reconciliation and authoritative readback.
- **Out of scope:** Partial cancellation, address changes, post-shipment exceptions.
- **Acceptance criteria:** Advanced fulfilment blocks execution; state changes invalidate the proposal; duplicate requests produce one cancellation; no success message precedes verification.
- **Required tests:** Eligible/ineligible; fulfilment race; expired approval; ownership mismatch; duplicate; pre/post-commit failures; verification mismatch.
- **Required evaluation cases:** Processing order, packed order, shipped order, already cancelled, state changes while awaiting approval and customer decline.
- **Verification commands:** `uv run pytest tests/domain/test_cancellation_rules.py tests/application/test_cancellations.py tests/workflows/option_a/test_cancellations.py -q`.
- **Risks:** Race between proposal and execution. Revalidation and expected order version are mandatory.

## A-06 — Route item problems and prohibited requests safely

- **Goal:** Ensure damaged, incorrect, missing, refund, replacement, exception and high-risk requests terminate in the correct safe path.
- **Dependencies:** A-03.
- **Likely files/modules:** `domain/preflight.py`, `domain/escalation.py`, `workflows/option_a/triage.py`, `web/issue_form.html`.
- **In scope:** Guided issue capture; explicit-human path; mandatory escalation precedence; safe refusal for harmless unsupported requests; proposal invalidation and write freeze.
- **Out of scope:** Image evidence, autonomous fraud decisions, refund/replacement tools, legal or safety advice.
- **Acceptance criteria:** All mandatory-escalation fixtures hand off; prohibited capabilities are absent; escalation freezes further automated writes; summaries contain only verified/minimum facts.
- **Required tests:** Rule precedence; each mandatory trigger; mixed intent; proposal invalidation; capability absence; redaction.
- **Required evaluation cases:** Damaged item, missing item, wrong item, explicit refund, threat, legal complaint, fraud language, unsupported shopping request and prompt injection.
- **Verification commands:** `uv run pytest tests/domain/test_preflight.py tests/workflows/option_a/test_triage.py -q`.
- **Risks:** Keyword-only risk detection can miss paraphrases. Record misses for Option E comparison while retaining obvious deterministic rules.

## A-07 — Produce the baseline release report

- **Goal:** Establish Option A's safety, quality, latency and operational baseline on a fixed dataset.
- **Dependencies:** A-01 through A-06.
- **Likely files/modules:** `evaluation/cases/option_a.jsonl`, `evaluation/scorers.py`, `evaluation/run.py`, `evaluation/load_test.py`, `observability/audit.py`, `observability/metrics.py`.
- **In scope:** Deterministic scorers; transition/tool sequence; duplicate-write checks; escalation recall; latency; synthetic workload; JSON and Markdown report; OpenTelemetry spans with redaction.
- **Out of scope:** LLM judge as release gate, real production claims, Langfuse requirement for Option A.
- **Acceptance criteria:** Zero critical privacy/auth/policy violations; 100% mandatory-escalation recall; zero duplicate writes; five concurrent conversations complete without concurrency errors; p95 results are reported honestly even when targets fail.
- **Required tests:** Scorer unit tests; golden report; trace redaction; deterministic rerun; load-test fixture isolation.
- **Required evaluation cases:** All happy paths, every mandatory escalation, all tool failures, cross-customer attempts, repeated approvals/messages and unknown writes.
- **Verification commands:** `uv run pytest tests/evaluation tests/observability -q`; `uv run python -m evaluation.run --variant option_a`; `uv run python -m evaluation.load_test --variant option_a --concurrency 5 --conversations 100`.
- **Risks:** Optimizing implementation to visible cases. Maintain held-out cases and report dataset version/hash.

## E-01 — Productionize the provider-neutral model boundary

- **Goal:** Integrate Gemini without allowing provider semantics into domain or action services.
- **Dependencies:** SPK-01; FND-01.
- **Likely files/modules:** `application/ports/model.py`, `adapters/models/gemini.py`, `adapters/models/fake.py`, `observability/model_runs.py`, `tests/contract/test_model_port.py`.
- **In scope:** Task-level methods (`interpret`, `draft_grounded_answer`, `draft_handoff`); budgets; cancellation/timeouts; Pydantic validation; one repair; hashes and metrics instead of full prompt retention.
- **Out of scope:** Model routing, LiteLLM, automatic fallback, business-tool execution.
- **Acceptance criteria:** Workflow-facing code imports only `ModelPort`; default tests use fake/replay; live calls are opt-in; exceeding call/token/cost budget returns a typed stop condition.
- **Required tests:** Adapter contract; fake/replay parity; invalid schema; refusal; timeout; budget exhaustion; telemetry redaction.
- **Required evaluation cases:** Reuse and freeze SPK-01 cases as regression cases.
- **Verification commands:** `uv run pytest tests/contract/test_model_port.py tests/adapters/test_gemini.py -q`; `RUN_LIVE_GEMINI=1 uv run pytest -m live_model tests/adapters/test_gemini_live.py -q`.
- **Risks:** Provider API/model change. Pin dependencies/model ID and keep provider payloads inside the adapter.

## E-02 — Route free text and clarify ambiguity

- **Goal:** Let customers express supported needs naturally while deterministic code chooses the permitted workflow.
- **Dependencies:** E-01; A-03; Option A workflow entry points.
- **Likely files/modules:** `workflows/option_e/state.py`, `workflows/option_e/graph.py`, `workflows/option_e/interpret.py`, `application/context_builder.py`, `tests/workflows/option_e/test_routing.py`.
- **In scope:** LangGraph entry/preflight; structured interpretation; non-authoritative entities; one-question clarification; turn limit; mandatory escalation before/after interpretation; deterministic route validation.
- **Out of scope:** Tool execution by the model, policy generation, action execution.
- **Acceptance criteria:** Natural-language requests reach an existing Option A use case; ambiguous consequential intent is clarified, never guessed; mandatory escalation overrides confidence; invalid model output repairs once then safely falls back/hands off.
- **Required tests:** Every graph edge; schema repair; clarification loop/limit; model timeout; explicit human; multi-intent; prompt injection; terminal states.
- **Required evaluation cases:** Paraphrases that Option A misses, vague order references, ambiguous return/cancel wording, mixed intents, high-risk language and adversarial instructions.
- **Verification commands:** `uv run pytest tests/workflows/option_e/test_routing.py -q`; `uv run python -m evaluation.run --variant option_e --suite routing`.
- **Risks:** Treating confidence as authority. Route only after deterministic required-field and capability checks.

## E-03 — Use bounded read tools for grounded policy and tracking answers

- **Goal:** Improve natural-language FAQ and order-status handling while restricting model-visible tools to two approved reads.
- **Dependencies:** E-02; A-01; A-02.
- **Likely files/modules:** `tools/gateway.py`, `workflows/option_e/read_tools.py`, `workflows/option_e/respond.py`, `knowledge/hybrid.py`, `tests/workflows/option_e/test_read_tools.py`.
- **In scope:** State-based tool exposure; `search_support_policy`; authorized `get_order_context`; evidence-to-answer validation; deterministic template fallback; optional pgvector experiment behind `KnowledgeRetriever`.
- **Out of scope:** Model access to ticket/return/cancel tools; conversation embeddings; answer from parametric memory; rerankers.
- **Acceptance criteria:** Anonymous cases cannot expose `get_order_context`; model cannot name or invoke write tools; every factual answer maps to approved evidence; missing/conflicting evidence hands off; vector retrieval remains disabled unless held-out recall improves materially.
- **Required tests:** Capability matrix; tool argument validation; injected customer ID ignored; source applicability; unsupported claim repair/fallback; tool timeout; retrieved-content injection.
- **Required evaluation cases:** FAQ paraphrases, policy conflict, tracking paraphrases, foreign order, stale shipment, malicious policy text and model-proposed prohibited call.
- **Verification commands:** `uv run pytest tests/tools/test_gateway.py tests/workflows/option_e/test_read_tools.py -q`; `uv run python -m evaluation.run --variant option_e --suite grounded_reads`.
- **Risks:** Tool calling may appear authoritative. Treat calls as proposals and validate independently before dispatch.

## E-04 — Start return and cancellation flows from natural language

- **Goal:** Add conversational entry and explanation to the proven action flows without changing their authority or write path.
- **Dependencies:** E-02; A-04; A-05.
- **Likely files/modules:** `workflows/option_e/actions.py`, `workflows/option_e/respond.py`, `application/context_builder.py`, `tests/workflows/option_e/test_actions.py`.
- **In scope:** Extract candidate order/item references; resolve against owned records; explain deterministic eligibility; render immutable proposal; accept only action-bound UI approval; call existing workflow-controlled action services; verified success or template fallback.
- **Out of scope:** Natural-language approval, LLM eligibility decisions, LLM-visible write tools, changed retry/reconciliation rules.
- **Acceptance criteria:** LLM output cannot set eligibility/approval/success; all writes pass unchanged A-04/A-05 safety tests; conversational paraphrases improve routing without introducing a critical violation.
- **Required tests:** Candidate resolution; hallucinated IDs; explanation contradiction; proposal rendering; ambiguous consent; stale state; duplicate click; model failure before/after verified write.
- **Required evaluation cases:** Natural return/cancel variants, wrong item/order candidate, ineligible dispute, exception request, model timeout, disconnect after write and repeated message.
- **Verification commands:** `uv run pytest tests/workflows/option_e/test_actions.py tests/application/test_returns.py tests/application/test_cancellations.py -q`; `uv run python -m evaluation.run --variant option_e --suite actions`.
- **Risks:** Generated prose diverging from proposal or result. Exact action fields come from deterministic state; use templates on mismatch.

## E-05 — Draft safe handoffs with deterministic fallback

- **Goal:** Reduce customer repetition by adding a grounded, redacted handoff summary without making escalation dependent on Gemini.
- **Dependencies:** E-01; A-03; A-06.
- **Likely files/modules:** `workflows/option_e/handoff.py`, `application/redaction.py`, `application/handoff.py`, `tests/workflows/option_e/test_handoff.py`.
- **In scope:** Draft from verified structured facts; AI-generated marker; schema/grounding/redaction validation; deterministic template fallback; existing ticket service.
- **Out of scope:** Full-transcript attachment, authoritative priority, specialist routing, model-generated allegations.
- **Acceptance criteria:** Required ticket fields are always present; unverified statements are excluded or labelled; model failure cannot block handoff; ticket count remains exactly one.
- **Required tests:** Unsupported summary claim; PII/secret redaction; invalid schema; model timeout; fallback parity; duplicate/unknown ticket result.
- **Required evaluation cases:** Item problem, identity failure, conflicting evidence, refund/replacement, high-risk language and long/noisy conversation.
- **Verification commands:** `uv run pytest tests/workflows/option_e/test_handoff.py tests/application/test_handoff.py -q`; `uv run python -m evaluation.run --variant option_e --suite handoff`.
- **Risks:** Compression can turn claims into facts. Preserve provenance/status per field and reject unsupported summaries.

## E-06 — Resume minimized workflow state and demonstrate governed memory

- **Goal:** Recover active cases safely and demonstrate explicit, correctable, expiring long-term facts without storing complete conversations.
- **Dependencies:** E-02; A-04; A-05; E-05.
- **Likely files/modules:** `adapters/db/checkpoints.py`, `memory/turn_buffer.py`, `memory/retained_facts.py`, `memory/retention.py`, `workflows/option_e/resume.py`, `tests/memory/`.
- **In scope:** PostgreSQL LangGraph checkpoint; minimized structured state; short-TTL turn buffer; session/state refresh on resume; opt-in retained fact with purpose/provenance/status/expiry; correction/deletion and index deletion propagation.
- **Out of scope:** Semantic conversation memory, preference profiles, cross-customer recall, transcript retention by default.
- **Acceptance criteria:** Restart resumes the last committed safe stage; pending unknown writes resume in reconciliation; expired/disputed/deleted facts are never injected; a customer correction supersedes the old fact; full conversations and prohibited fields are absent from durable memory.
- **Required tests:** Crash at every consequential stage; session expiry; state/customer mismatch; TTL deletion; correction; deletion receipt; prohibited-field rejection; checkpoint pruning.
- **Required evaluation cases:** Resume during clarification, awaiting approval, unknown write and verified-write/undelivered-message; repeat unresolved case with and without permitted retained fact.
- **Verification commands:** `uv run pytest tests/memory tests/workflows/option_e/test_resume.py -q`.
- **Risks:** Framework checkpoints accidentally retaining raw messages. Assert serialized checkpoint keys and size in tests.

## E-07 — Compare, load-test and package the portfolio release

- **Goal:** Decide from evidence whether Option E is justified over Option A and produce a reproducible deployed demonstration.
- **Dependencies:** A-07; E-01 through E-06.
- **Likely files/modules:** `evaluation/cases/shared.jsonl`, `evaluation/compare.py`, `evaluation/load_test.py`, `observability/langfuse.py`, `Dockerfile`, `README.md`, `docs/evaluation_report.md`.
- **In scope:** Identical held-out cases and mock states; deterministic primary scoring; Langfuse experiment metadata; cost/latency; five-conversation concurrency; Docker build; explicit limitations and synthetic-data disclosure.
- **Out of scope:** Claiming production readiness, LLM judge as gate, automatic model promotion, multi-agent system, Kubernetes.
- **Acceptance criteria:** Option E meets at least 90% safe/correct resolution, 100% mandatory-escalation recall, zero critical violations, at least 95% valid tool selection/arguments, p95 workflow below 15 seconds and average variable cost at or below $0.05; report shows whether natural-language completion materially beats Option A. If it does not, Option A remains the recommended MVP.
- **Required tests:** Full regression suite; Docker smoke test; migration on empty database; trace redaction; deterministic evaluator repeatability; live-model tests separated from CI-default tests.
- **Required evaluation cases:** Shared happy paths, paraphrases, ambiguity, prompt/retrieval injection, cross-customer access, all mandatory handoffs, every tool failure, retries, concurrency and crash recovery.
- **Verification commands:** `uv run pytest`; `uv run python -m evaluation.run --variant option_a`; `uv run python -m evaluation.run --variant option_e`; `uv run python -m evaluation.compare`; `uv run python -m evaluation.load_test --concurrency 5 --conversations 100`; `docker build -t support-agent-mvp .`.
- **Risks:** Free-tier rate limits distort load tests and customer data would be inappropriate. Use replay/fakes for repeatable load evidence, a separately labelled live sample for model latency/cost, and synthetic data only.

---

## Release gates

No slice may bypass these gates to satisfy latency, cost or demo convenience:

- No protected read before trusted session and ownership checks.
- No return/cancellation without fresh deterministic eligibility and exact explicit approval.
- No blind retry of an unknown state-changing outcome.
- No success or handoff claim before independent readback verification.
- No LLM authority over identity, policy applicability, eligibility, approval, execution, verification, escalation or terminal state.
- No default long-term transcript storage or customer-conversation embeddings.
- Any critical privacy, authorization or policy violation blocks release.

## Suggested three-to-four-week sequence

| Order | Sessions | Outcome |
|---|---|---|
| 1 | SPK-01, SPK-02 | Highest technical risks measured before commitment |
| 2 | FND-01, A-01, A-02, A-03 | Runnable public, protected-read and handoff paths |
| 3 | A-04, A-05, A-06, A-07 | Complete deterministic baseline and benchmark |
| 4 | E-01, E-02, E-03 | Bounded Gemini interpretation and read-tool value |
| 5 | E-04, E-05, E-06 | Conversational actions, handoff and governed memory |
| 6 | E-07 | Comparative release evidence and deployable portfolio package |

If time is constrained, stop after E-03 and compare natural-language routing/grounded reads against Option A. Do not compress the schedule by removing authorization, approval, idempotency, verification, escalation or evaluation work.
