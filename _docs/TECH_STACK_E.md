# Option E — Technology Stack

## Decision

Reuse Option A's API, database, deterministic controls, UI, mock tools and evaluation dataset. Add LangGraph only for the bounded workflow, a thin provider-neutral model port, Gemma tool schemas with local Pydantic validation, PostgreSQL hybrid retrieval and LLM-specific tracing.

This remains a deterministic workflow containing LLM steps—not a free-running agent.

## Selected models

### Runtime LLM

**Primary model:** **Gemma 4 26B A4B Instruct** through a hosted inference endpoint. Keep the provider's concrete model ID in configuration rather than workflow code.

Use it for:

- Intent classification and non-authoritative entity extraction.
- Bounded clarification-question generation.
- Model-visible read-tool requests in allowlisted workflow states.
- Grounded policy and order-status explanations.
- Drafting redacted escalation summaries.

Configuration:

- Disable or minimize thinking for routine bounded calls; enable it only if evaluation proves a tool-selection or accuracy gain within the latency budget.
- Pydantic-derived JSON tool declarations, followed by local validation and one bounded repair attempt.
- Low output-token limits per node; do not send the full conversation by default.
- No automatic model upgrade, cross-provider fallback or model-selected model routing.
- The model may propose calls only to approved read tools. Authentication, eligibility, approval, writes, verification and escalation remain deterministic.

**Why:** Gemma 4 supports function calling and the 26B A4B variant is Google's recommended starting point for many applications with lower resource requirements. It is only an initial candidate: release depends on the approved safety, quality, latency and cost evaluation.

### Smaller model candidate

**Candidate:** **Gemma 4 12B Instruct**, tested offline if 26B A4B meets quality thresholds but misses the latency, cost or hosting constraints.

Do not silently route difficult cases between models. If 26B A4B misses a quality or safety threshold, evaluate a stronger suitable model through the same `ModelPort`; promote a replacement only after fixed-dataset comparison. The replacement need not be from the Gemma family.

### Embedding model

**Initial candidate:** **EmbeddingGemma 308M** for the optional pgvector side of hybrid policy retrieval.

Keep embeddings only if held-out paraphrase tests materially outperform PostgreSQL full-text search. Embeddings are not used for customer-conversation memory.

## Layer decisions

### 1. API

**Options:** FastAPI; Flask; Django + Django REST Framework.

**Recommend:** **FastAPI** with SSE token/event streaming.

- **Requirement:** `<2s` first useful response, typed tool endpoints, chat streaming and shared infrastructure with Option A.
- **Why simpler is insufficient:** Flask needs more manual async/schema work. Django is excessive for one bounded chat workflow.
- **Burden:** Async cancellation, disconnect handling and back-pressure.
- **Replaceability:** Medium; routes must call framework-neutral application services.
- **Defer:** WebSockets, GraphQL and multi-channel APIs.

### 2. Agent/workflow orchestration

**Options:** Plain Python state machine; LangGraph `StateGraph`; an OpenAI/LangChain autonomous agent loop.

**Recommend:** **LangGraph `StateGraph` with explicit nodes and conditional edges**, while deterministic domain services retain authority.

- **Requirement:** Demonstrate orchestration, bounded tool calling, clarification loops, checkpoint/resume and human approval without surrendering control.
- **Why simpler is insufficient:** Plain Python remains viable but would duplicate the already-large transition engine and make graph/checkpoint inspection less visible in the portfolio. An autonomous loop conflicts with the approved architecture.
- **Burden:** LangGraph concepts, checkpoint schema, framework upgrades and trace interpretation.
- **Replaceability:** Medium if each node invokes framework-neutral services and the typed `WorkflowState` remains application-owned.
- **Defer:** Supervisor-worker graphs, subagents, dynamic plans and parallel writes.

### 3. Model-provider abstraction

**Options:** Direct hosted-inference SDK throughout the code; LiteLLM; a thin application `ModelPort` with provider adapters.

**Recommend:** **A thin typed `ModelPort`, a `GemmaAdapter` for the selected hosted endpoint, and a deterministic fake adapter for tests**.

- **Requirement:** Provider abstraction, tool calling, cost/latency accounting and repeatable tests.
- **Why simpler is insufficient:** Scattered provider calls create lock-in and inconsistent retry/telemetry policy. LiteLLM adds a broad dependency and proxy behaviours before a second provider is actually needed.
- **Burden:** Maintain a small common request/result contract and explicitly handle unsupported provider features.
- **Replaceability:** Low; add or replace one adapter rather than changing workflows.
- **Defer:** A second live provider, LiteLLM proxy, automatic model routing and fallback across vendors.

The port should expose task-level methods such as `interpret`, `draft_grounded_answer` and `draft_handoff`; the adapter owns Gemma chat templates, tool-call parsing and provider-specific parameters. It must not pretend every provider has identical low-level semantics.

### 4. Structured outputs

**Options:** Prompted JSON plus parsing; Instructor; Gemma JSON tool declarations backed by Pydantic.

**Recommend:** **Pydantic v2 as the authoritative boundary**, generating Gemma-compatible tool declarations, parsing proposed function calls locally, and allowing at most one bounded repair.

- **Requirement:** Model output is untrusted, typed and incapable of directly setting authoritative state.
- **Why simpler is insufficient:** Prompt-only JSON is too fragile for routing and tool arguments. Instructor adds another abstraction before a second provider requires it. Gemma output remains untrusted even when it matches the declared schema.
- **Burden:** Schema compatibility, refusal handling and validation/repair paths.
- **Replaceability:** Low because Pydantic models remain the source of truth.
- **Defer:** Grammar-constrained local inference and automatic multi-repair loops.

### 5. Operational database

**Options:** SQLite; PostgreSQL; document database.

**Recommend:** **The same PostgreSQL, SQLAlchemy 2 and Alembic foundation as Option A**.

- **Requirement:** Authoritative cases, approvals, idempotency, audits, model-run metadata and reliable comparison with Option A.
- **Why simpler is insufficient:** SQLite weakens concurrent writes/checkpointing. A document store does not improve the relational invariants.
- **Burden:** Migrations, connection management, backups and retention jobs.
- **Replaceability:** High; keep repositories and domain models separate from ORM records.
- **Defer:** Separate analytics database, event bus and read replicas.

### 6. Vector retrieval

**Options:** PostgreSQL full-text only; PostgreSQL + pgvector hybrid retrieval; managed vector database.

**Recommend:** **Hybrid retrieval in the existing PostgreSQL database: full-text/`pg_trgm` plus pgvector**, with deterministic version/effective-date filters and an optional EmbeddingGemma adapter.

- **Requirement:** The LLM must retrieve approved policy evidence and improve handling of natural-language paraphrases while preserving source/version provenance.
- **Why simpler is insufficient:** Full-text search remains the fallback and baseline, but may miss semantically equivalent wording. A separate vector service is unjustified for the small corpus.
- **Burden:** Chunking, embedding refresh, index tuning, hybrid-score evaluation and deletion propagation.
- **Replaceability:** Medium behind `KnowledgeRetriever`; retain canonical document/chunk IDs independent of vector technology.
- **Defer:** Rerankers, query rewriting, multimodal retrieval and a separate vector database.

If held-out evaluation shows no material retrieval gain, disable vector search and retain full-text search only.

### 7. Short-term state/checkpointing

**Options:** LangGraph in-memory saver; LangGraph SQLite saver; LangGraph PostgreSQL saver.

**Recommend:** **LangGraph PostgreSQL checkpointer in the existing database**, using a UUID case/thread ID and storing only minimized structured state.

- **Requirement:** Multi-turn clarification, human approval, restart recovery and uncertain-write reconciliation.
- **Why simpler is insufficient:** In-memory checkpoints disappear on restart. SQLite creates a second persistence path and differs from deployment.
- **Burden:** Checkpoint pruning, schema migration and careful prevention of raw transcript accumulation.
- **Replaceability:** Medium; checkpoint APIs are framework-specific, while domain state remains in application tables.
- **Defer:** Redis checkpointing and cross-region state replication.

The LangGraph checkpoint is execution state, not business truth. PostgreSQL domain tables remain authoritative for approvals, actions and audit.

### 8. Long-term memory

**Options:** LangGraph Store/vector memory; Mem0 or similar memory service; governed structured facts in PostgreSQL.

**Recommend:** **Governed structured facts in `retained_case_summaries`, disabled by default**, with explicit purpose, provenance, expiry, verification, correction and deletion.

- **Requirement:** Demonstrate memory while complying with the approved data-minimisation model and prohibition on default transcript retention.
- **Why simpler is insufficient:** No implementation would fail to demonstrate memory governance. Automatic semantic memory introduces privacy and correction/deletion risks without an MVP use case.
- **Burden:** Policy enforcement, TTL jobs, customer correction/deletion flow and deletion propagation tests.
- **Replaceability:** Low; ordinary relational facts and service interfaces are portable.
- **Defer:** Personal preferences, automatic model-written memory, cross-customer memory, embeddings of conversations and LangGraph cross-thread Store.

### 9. Front end

**Options:** Streamlit; FastAPI + Jinja2/HTMX/vanilla JavaScript; React/Next.js.

**Recommend:** **Reuse Option A's Jinja2 + HTMX/vanilla JavaScript UI**, adding SSE rendering for model tokens/events.

- **Requirement:** Fair A/E comparison, explicit approvals and a portfolio-quality website chat.
- **Why simpler is insufficient:** Streamlit changes the interaction model and weakens the comparison. React adds a separate application stack without solving a core risk.
- **Burden:** Streaming event handling, reconnect behaviour and safe rendering.
- **Replaceability:** High through stable HTTP/SSE contracts.
- **Defer:** Separate SPA, voice and mobile clients.

### 10. Observability

**Options:** OpenTelemetry only; Langfuse only; OpenTelemetry plus Langfuse Cloud.

**Recommend:** **OpenTelemetry for end-to-end application traces plus hosted Langfuse for LLM/retrieval/prompt/token/cost traces**.

- **Requirement:** Observe every transition and tool call, compare A/E latency/cost, debug model behaviour and show grounded evidence.
- **Why simpler is insufficient:** OpenTelemetry alone lacks convenient prompt/model/evaluation views. Langfuse alone should not replace operational database/API metrics.
- **Burden:** Two telemetry destinations, context propagation, redaction and sampling. Do not send credentials, complete profiles or default full conversations.
- **Replaceability:** Low-to-medium because Langfuse accepts OpenTelemetry and instrumentation should be wrapped.
- **Defer:** Self-hosted Langfuse/Grafana, full prompt management in production and alert paging.

### 11. Evaluation

**Options:** Custom evaluator only; Ragas/DeepEval; custom evaluator plus Langfuse datasets/experiments.

**Recommend:** **The shared deterministic JSONL evaluation harness plus Langfuse dataset experiments** for prompt/model variants.

- **Requirement:** Prove Option E improves natural-language handling while retaining 100% mandatory-escalation recall, zero critical violations and the cost/latency limits.
- **Why simpler is insufficient:** Manual traces cannot support a baseline comparison. Generic RAG scores alone cannot verify authorization, approval or duplicate-write invariants.
- **Burden:** Dataset versioning, deterministic primary scorers, trace-to-case correlation and review of regressions.
- **Replaceability:** Low; exportable cases, expected outcomes and scores are independent of the UI.
- **Defer:** LLM-as-judge as a release gate. It may be a secondary diagnostic after calibration against human labels.

Primary scorers should remain code-based: final state, tool sequence, arguments, evidence IDs, escalation reason, unsupported claims, latency, tokens and cost.

### 12. Testing

**Options:** pytest only; pytest plus Hypothesis; pytest/Hypothesis plus contract, replay and Testcontainers tests.

**Recommend:** **pytest, pytest-asyncio, HTTPX, Hypothesis, Testcontainers/Docker Compose, and a deterministic model fake/replay fixture**.

- **Requirement:** State-machine invariants, model schema failures, prompt injection, tool timeouts, uncertain writes and provider-independent tests.
- **Why simpler is insufficient:** Live-model-only tests are slow, costly and nondeterministic; example-only tests miss action sequences and malformed outputs.
- **Burden:** Curated recordings/fakes, Docker integration tests and separate optional live-model test markers.
- **Replaceability:** Low.
- **Defer:** Large browser farms, continuous fuzzing infrastructure and production shadow traffic.

### 13. Packaging

**Options:** pip requirements; Poetry; uv + `pyproject.toml`/`uv.lock`.

**Recommend:** **uv with one `pyproject.toml` and locked dependency groups** for core, Option A, Option E and development.

- **Requirement:** Reproducible comparison and fast one-developer setup.
- **Why simpler is insufficient:** Unlocked requirements risk non-reproducible model/framework behaviour.
- **Burden:** Minimal lockfile maintenance and deliberate dependency upgrades.
- **Replaceability:** Low.
- **Defer:** Multi-package monorepo tooling and package publication.

### 14. Deployment

**Options:** Single Docker service on Render; Railway; Google Cloud Run. Use provider-managed PostgreSQL or a separate managed PostgreSQL service.

**Recommend:** **One Dockerized FastAPI/LangGraph service plus one managed PostgreSQL database on a simple PaaS**, using hosted Gemma inference and hosted Langfuse as the only additional external services.

- **Requirement:** 100 conversations/day, five concurrent conversations, `<15s` tool workflows and low operational burden.
- **Why simpler is insufficient:** Local-only deployment is not a portfolio deployment. Function-only hosting complicates streaming, connection pooling and checkpoints.
- **Burden:** Secrets, migrations, health checks, hosted-inference quotas and telemetry egress.
- **Replaceability:** Medium; the OCI image, provider adapter and standard database keep the host portable.
- **Defer:** Celery workers, Redis, Kubernetes, autoscaling policies, multi-region failover and self-hosted model inference.

## Recommended MVP stack

- Python 3.12+
- FastAPI + Uvicorn + SSE
- LangGraph `StateGraph` with explicit bounded nodes
- Thin `ModelPort`; hosted-inference `GemmaAdapter`; deterministic fake adapter
- Gemma 4 26B A4B Instruct as the initial runtime LLM
- Gemma 4 12B Instruct only as an evaluated latency/cost downgrade candidate
- EmbeddingGemma 308M only for evaluated hybrid policy retrieval
- Pydantic v2 + Gemma JSON tool declarations + local validation
- PostgreSQL + SQLAlchemy 2 + Alembic
- PostgreSQL full-text/`pg_trgm` + pgvector hybrid retrieval
- LangGraph PostgreSQL checkpointer with minimized state
- Governed relational case facts; no automatic semantic long-term memory
- Jinja2 + HTMX/vanilla JavaScript
- OpenTelemetry + hosted Langfuse
- Shared JSONL evaluation harness + Langfuse experiments
- pytest, pytest-asyncio, HTTPX, Hypothesis, Testcontainers and model replay/fakes
- uv + `pyproject.toml` + `uv.lock`
- One Docker container on a PaaS + managed PostgreSQL

## Rejected components for the MVP

- ReAct/AgentExecutor-style autonomous loops and unrestricted model tool access.
- Supervisor-worker or multi-agent systems.
- FunctionGemma as the main conversational model; it is a small foundation for specialized function-calling fine-tuning, not the general assistant selected here.
- Self-hosted GPU inference for the MVP.
- LiteLLM proxy before a second provider is required.
- Mem0, Zep or automatic cross-session memory.
- Pinecone, Weaviate, Qdrant, Elasticsearch or another retrieval service.
- Redis, Celery, Kafka, Temporal and Airflow.
- A second operational database or data warehouse.
- Kubernetes, microservices and self-hosted Langfuse/Grafana.
- LLM-as-judge as the sole or release-blocking evaluator.

## Reconsider after measurable thresholds

| Component | Reconsider when |
|---|---|
| Replace Gemma 4 26B A4B | After prompt/schema tuning it misses any release threshold: `<90%` safe/correct autonomous resolution, `<100%` mandatory-escalation recall, any critical privacy/auth/policy violation, `<95%` valid tool selection and arguments, `>15s` p95 workflow latency, or `>$0.05` average variable cost |
| Evaluate Gemma 4 12B | 26B A4B meets safety and quality thresholds but misses latency, cost or hosting constraints |
| Second provider/LiteLLM | Provider outages or price/quality tests show material benefit, or at least two production providers must be supported |
| Self-hosted Gemma inference | Hosted spend, latency or data-residency needs justify GPU operations and projected utilization makes the infrastructure economical |
| Redis | More than 50 concurrent active cases, checkpoint p95 exceeds 100 ms, or distributed throttling is required |
| Worker queue | More than 5% of workflows exceed 30 seconds or must continue after the HTTP/SSE connection closes |
| Separate vector database | More than 1 million active chunks, retrieval p95 exceeds 200 ms after tuning, or vector load harms transactional SLOs |
| Reranker | Hybrid retrieval misses materially reduce grounded-answer accuracy and offline tests show a significant gain within cost/latency limits |
| Temporal | Workflows last hours/days and durable timers/external signals become frequent operational requirements |
| Supervisor-worker | Multiple independent domains exist and over 20% of cases need cross-domain/parallel investigation, with held-out evidence of better outcomes |
| Self-hosted observability | Trace volume/cost, data residency or contractual restrictions make hosted telemetry unacceptable |
| Kubernetes | At least three independently scaled services, multi-region operation or explicit availability objectives justify it |

## Primary documentation

- [FastAPI features](https://fastapi.tiangolo.com/features/)
- [LangGraph workflows and agents](https://docs.langchain.com/oss/python/langgraph/workflows-agents)
- [LangGraph persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Gemma getting started](https://ai.google.dev/gemma/docs/get_started)
- [Gemma 4 function calling](https://ai.google.dev/gemma/docs/capabilities/text/function-calling-gemma4)
- [Gemma model selection](https://ai.google.dev/gemma/docs/core)
- [EmbeddingGemma](https://ai.google.dev/gemma/docs/embeddinggemma)
- [FunctionGemma model card](https://ai.google.dev/gemma/docs/functiongemma/model_card)
- [PostgreSQL full-text search](https://www.postgresql.org/docs/current/textsearch.html)
- [pgvector](https://github.com/pgvector/pgvector)
- [OpenTelemetry Python](https://opentelemetry.io/docs/languages/python/)
- [Langfuse documentation](https://langfuse.com/docs)
- [pytest](https://docs.pytest.org/en/stable/)
- [Hypothesis](https://hypothesis.readthedocs.io/en/latest/)
- [uv](https://docs.astral.sh/uv/)
