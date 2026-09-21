# Option A — Technology Stack

## Decision

Build Option A as a deterministic Python web application with an explicit state machine, PostgreSQL, typed contracts and a shared evaluation harness. It must remain a credible control group: no LLM, agent framework, embeddings or semantic memory.

## Selected model

**None.** Option A deliberately makes zero LLM or embedding API calls. This preserves a valid deterministic baseline for measuring whether Option E's additional cost and nondeterminism provide material value.

## Layer decisions

### 1. API

**Options:** FastAPI; Flask; Django + Django REST Framework.

**Recommend:** **FastAPI** with async endpoints and Server-Sent Events (SSE) only where streaming improves perceived latency.

- **Requirement:** Typed mock-tool APIs, website chat, `<2s` first response and strict request validation.
- **Why simpler is insufficient:** Flask can work, but requires more manual schema/OpenAPI/async wiring. Django adds an unnecessary application framework and admin/auth surface.
- **Burden:** ASGI lifecycle, async database sessions and exception handling.
- **Replaceability:** Medium; keep business workflows independent of FastAPI route objects.
- **Defer:** WebSockets, GraphQL and public API versioning.

### 2. Workflow orchestration

**Options:** Explicit Python state machine; LangGraph; Temporal.

**Recommend:** **An explicit Python transition table and workflow service** using enums, pure transition functions and repository interfaces.

- **Requirement:** The approved architecture has explicit transitions, terminal states, retries, reconciliation and write gates.
- **Why simpler is insufficient:** Logic spread across route handlers cannot reliably enforce invariants or resume after failure. A formal state machine is the simplest adequate design.
- **Burden:** Transition definitions, guards and state-machine tests must be maintained.
- **Replaceability:** Medium if transitions call domain services through interfaces; high if framework details leak into rules.
- **Defer:** LangGraph and Temporal. Option A must not acquire an agent framework merely for visual appeal.

### 3. Model-provider abstraction

**Options:** No model dependency; a custom provider protocol; LiteLLM.

**Recommend:** **No model dependency in Option A**.

- **Requirement:** Option A is the deterministic experimental baseline.
- **Why simpler is insufficient:** It is not—absence of an LLM is the required behaviour. Provider abstraction is demonstrated in Option E.
- **Burden:** None.
- **Replaceability:** Not applicable.
- **Defer:** All model SDKs, token accounting and prompt management.

### 4. Structured outputs and validation

**Options:** Pydantic v2; dataclasses + manual validation; Marshmallow.

**Recommend:** **Pydantic v2 models and generated JSON Schema** for API, workflow events and mock-tool contracts.

- **Requirement:** Typed inputs/outputs, deny-by-default fields and consistent production/simulation contracts.
- **Why simpler is insufficient:** Dataclasses type values but do not provide strict runtime validation and schema generation without more code.
- **Burden:** Schema versioning and validation-error mapping.
- **Replaceability:** Low-to-medium; isolate models at boundaries and map them to domain types.
- **Defer:** A schema registry.

### 5. Operational database

**Options:** SQLite; PostgreSQL; document database.

**Recommend:** **PostgreSQL with SQLAlchemy 2 and Alembic**.

- **Requirement:** Durable state, optimistic concurrency, approvals, idempotency uniqueness, audit events and five concurrent conversations.
- **Why simpler is insufficient:** SQLite could run the demo, but weakens the concurrency/idempotency story and would create a deployment migration before Option E. A document database adds no benefit to relational invariants.
- **Burden:** Connection pooling, migrations, backups and a managed database in deployment.
- **Replaceability:** High; database semantics are central. SQLAlchemy reduces provider coupling, not SQL coupling.
- **Defer:** Read replicas, partitioning, event sourcing and a data warehouse.

### 6. Knowledge retrieval

**Options:** In-memory keyword matching; PostgreSQL full-text search plus `pg_trgm`; a vector database.

**Recommend:** **PostgreSQL full-text search with `pg_trgm` fallback** over approved, versioned FAQs and policies.

- **Requirement:** Option A specifies keyword/conventional search and must detect no-result or conflicting-source cases.
- **Why simpler is insufficient:** Plain substring matching handles spelling and phrasing poorly and offers weak ranking. Vector search would contaminate the non-LLM baseline.
- **Burden:** Search indexes, ranking tests and content-version filters.
- **Replaceability:** Low; expose a `KnowledgeRepository` interface and preserve source IDs/hashes.
- **Defer:** Embeddings, reranking and external search services.

### 7. Short-term state and checkpointing

**Options:** Process memory; Redis; PostgreSQL case/checkpoint tables.

**Recommend:** **PostgreSQL as the sole checkpoint store**, with transactional updates and an optimistic `version` column.

- **Requirement:** Crash recovery, session resumption, proposal expiry and unknown-write reconciliation.
- **Why simpler is insufficient:** In-memory state is lost on restart. Redis is an unjustified second datastore at five concurrent conversations.
- **Burden:** State serialization, retention cleanup and concurrency-conflict handling.
- **Replaceability:** Medium behind a `WorkflowRepository`.
- **Defer:** Redis and distributed locks.

### 8. Long-term memory

**Options:** No durable case facts; structured PostgreSQL facts; vector/semantic memory.

**Recommend:** **A disabled-by-default `retained_case_summaries` PostgreSQL table** with explicit purpose, provenance, verification status, expiry and correction/deletion support.

- **Requirement:** The information model allows narrow case memory but forbids complete-conversation retention by default.
- **Why simpler is insufficient:** Keeping nothing would prevent demonstrating governed retention and repeat-case handling. Free-form/vector memory would violate minimisation.
- **Burden:** TTL cleanup, correction/deletion endpoints and audit receipts.
- **Replaceability:** Low because it uses ordinary relational records.
- **Defer:** Personal preferences, cross-session semantic recall and automatic memory writes.

### 9. Front end

**Options:** Streamlit; FastAPI + Jinja2/HTMX and small vanilla JavaScript; React/Next.js.

**Recommend:** **Jinja2 + HTMX/vanilla JavaScript served by FastAPI**, with accessible buttons/forms and SSE for progress.

- **Requirement:** Website live chat, explicit confirmation controls and a realistic portfolio UX.
- **Why simpler is insufficient:** Streamlit is quick but obscures exact browser/API interaction and approval controls. React adds a second application toolchain without enough MVP value.
- **Burden:** Modest HTML/CSS/JavaScript and accessibility testing.
- **Replaceability:** High because the API remains independent.
- **Defer:** Separate SPA, design system and additional channels.

### 10. Observability

**Options:** Plain logs; structured JSON logs plus OpenTelemetry; self-hosted Grafana stack.

**Recommend:** **Structured JSON logging plus OpenTelemetry spans/metrics**, exported to console locally and an OTLP-compatible hosted backend in the deployed demo.

- **Requirement:** Correlation IDs, transition/tool/audit traces, latency percentiles and failure diagnosis.
- **Why simpler is insufficient:** Plain text logs cannot reliably reconstruct a case or compare Option A with E.
- **Burden:** Trace propagation, redaction, sampling and dashboard configuration.
- **Replaceability:** Low because OpenTelemetry is vendor-neutral.
- **Defer:** Self-hosted collectors, Loki/Tempo/Prometheus clusters and on-call paging.

### 11. Evaluation

**Options:** Custom Python evaluator; Ragas/DeepEval; vendor-hosted evaluation suite.

**Recommend:** **A custom, versioned JSONL scenario set and Python evaluation runner**, producing JSON/CSV metrics for correctness, escalation recall, privacy violations, duplicates, latency and cost.

- **Requirement:** Both options must run against identical cases, mock states and fault injections.
- **Why simpler is insufficient:** Ad hoc manual demos cannot prove the acceptance criteria. Generic LLM evaluators do not fit deterministic state/action assertions.
- **Burden:** Ground-truth fixtures, scorer maintenance and reproducible reports.
- **Replaceability:** Low; dataset and metric contracts remain useful with any runner.
- **Defer:** LLM judges and production-traffic evaluation.

### 12. Testing

**Options:** `unittest`; `pytest`; `pytest` plus Hypothesis and Testcontainers.

**Recommend:** **pytest + pytest-asyncio + HTTPX + Hypothesis**, with PostgreSQL integration tests through Testcontainers or Docker Compose.

- **Requirement:** Transition coverage, schema/tool contracts, concurrency, retries, expiry and zero duplicate writes.
- **Why simpler is insufficient:** Example-based unit tests alone are weak for state-machine invariants and retry/idempotency edge cases.
- **Burden:** Docker-dependent integration tests and careful property definitions.
- **Replaceability:** Low; tests are mostly framework-independent.
- **Defer:** Browser-farm testing and large-scale chaos infrastructure.

### 13. Packaging

**Options:** pip + requirements files; Poetry; uv + `pyproject.toml`/`uv.lock`.

**Recommend:** **uv with `pyproject.toml` and committed `uv.lock`**, plus Ruff and mypy/pyright in CI.

- **Requirement:** Reproducible one-developer setup and identical local/CI/container dependencies.
- **Why simpler is insufficient:** An unpinned requirements file weakens reproducibility. Poetry duplicates capabilities without a project-specific advantage.
- **Burden:** Minimal; contributors need uv or the container.
- **Replaceability:** Low; standard project metadata remains portable.
- **Defer:** Publishing packages to PyPI and a monorepo build system.

### 14. Deployment

**Options:** Single Docker service on Render; Railway; Google Cloud Run. Database options are provider-managed PostgreSQL or a separate managed PostgreSQL service.

**Recommend:** **One Dockerized FastAPI service plus one managed PostgreSQL database on a simple PaaS**; Render is the default portfolio target, with configuration kept portable.

- **Requirement:** Deployable MVP at 100 conversations/day and five concurrent conversations.
- **Why simpler is insufficient:** Local-only deployment does not demonstrate operability. Serverless functions are awkward for streaming and durable request workflows.
- **Burden:** Secrets, migrations, health checks, TLS and basic backup settings.
- **Replaceability:** Medium; the OCI image and standard PostgreSQL connection string preserve portability.
- **Defer:** Kubernetes, service mesh, CDN, multi-region deployment, background workers and autoscaling design.

## Recommended MVP stack

- **LLM model: none**
- Python 3.12+
- FastAPI + Uvicorn
- Pydantic v2
- Explicit Python state machine and domain services
- PostgreSQL + SQLAlchemy 2 + Alembic
- PostgreSQL full-text search + `pg_trgm`
- Jinja2 + HTMX/vanilla JavaScript + SSE
- Structured JSON logs + OpenTelemetry
- Custom JSONL evaluation harness
- pytest, pytest-asyncio, HTTPX, Hypothesis and Testcontainers/Docker Compose
- uv + `pyproject.toml` + `uv.lock`
- One Docker container on a PaaS + managed PostgreSQL

## Rejected components for the MVP

- Any LLM SDK, agent framework or embedding model.
- LangGraph, Temporal, Celery, Redis, Kafka and Airflow.
- Pinecone, Weaviate, Qdrant, Elasticsearch or another separate retrieval service.
- Vector or free-form customer memory.
- React/Next.js unless UI quality becomes the project's main constraint.
- Kubernetes, microservices and self-hosted observability infrastructure.

## Reconsider after measurable thresholds

| Component | Reconsider when |
|---|---|
| Redis | More than 50 concurrent active cases, database checkpoint p95 exceeds 100 ms, or distributed rate limiting becomes necessary |
| Background queue | More than 5% of workflows need work beyond 30 seconds or beyond the HTTP request lifecycle |
| Temporal | Workflows routinely last hours/days and require external signals, durable timers and many retries |
| Separate search service | Approved corpus exceeds roughly 100,000 documents or PostgreSQL search p95 exceeds 200 ms under target load |
| Read replicas/partitioning | Sustained database CPU exceeds 70% or read p95 breaches the latency budget after query/index tuning |
| Kubernetes | At least three independently scaled services, multi-region requirements or an explicit availability SLO justify it |

## Primary documentation

- [FastAPI features](https://fastapi.tiangolo.com/features/)
- [PostgreSQL full-text search](https://www.postgresql.org/docs/current/textsearch.html)
- [PostgreSQL trigram matching](https://www.postgresql.org/docs/current/pgtrgm.html)
- [OpenTelemetry Python](https://opentelemetry.io/docs/languages/python/)
- [pytest](https://docs.pytest.org/en/stable/)
- [Hypothesis](https://hypothesis.readthedocs.io/en/latest/)
- [uv](https://docs.astral.sh/uv/)
