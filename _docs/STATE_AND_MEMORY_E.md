# Option E — State, Knowledge and Memory

## Decision

Option E uses the same deterministic authoritative state as Option A and adds **bounded, ephemeral LLM context**. The model may derive intent, entities, clarification needs and draft text, but its output is non-authoritative until schema validation and deterministic checks succeed.

The LLM has no free-form long-term memory. Complete conversations are not retained as long-term memory by default.

## Data classes

| Class | Examples | Authority | Default lifetime |
|---|---|---|---|
| Ephemeral model context | Recent redacted turns, current task, permitted tool results | None by itself | One model call or active case |
| Model-derived state | Intent candidate, entity candidate, summary draft, risk signals | Non-authoritative | Active case; discard when resolved unless approved for handoff |
| Authoritative state | Session subject, order/tool results, eligibility, approval, verified write | Trusted services/workflow | Case/audit retention |
| Knowledge evidence | Approved policy passages and metadata | Approved repository | Referenced by version/hash |
| Operational memory | Current stage, proposals, idempotency, reconciliation state | Deterministic workflow | Active case + recovery window |
| Long-term case memory | Minimal structured case facts | Application | Exceptional, criteria-bound and expiring |

## Typed workflow state model

```ts
type Intent =
  | "FAQ"
  | "TRACK_ORDER"
  | "START_RETURN"
  | "CANCEL_ORDER"
  | "ITEM_PROBLEM"
  | "HUMAN_REQUEST"
  | "UNSUPPORTED";

type Stage =
  | "START"
  | "INTERPRETING"
  | "CLARIFYING"
  | "AUTH_REQUIRED"
  | "ORDER_SELECTION"
  | "CONTEXT_LOADING"
  | "ELIGIBILITY_CHECK"
  | "AWAITING_CONFIRMATION"
  | "EXECUTING"
  | "RECONCILING"
  | "ESCALATING"
  | "RESOLVED"
  | "ESCALATED"
  | "ABANDONED"
  | "FAILED_SAFE";

type DerivedInterpretation = {
  intent_candidate: Intent;
  confidence: number; // 0..1; routing aid, never authorization
  order_ref_candidate?: string;
  item_ref_candidates: string[];
  requested_outcome?: string;
  risk_signals: Array<"FRAUD" | "THREAT" | "LEGAL" | "SAFETY">;
  ambiguity_codes: string[];
  source_turn_ids: string[];
  model_run_id: string;
  created_at: string;
};

type IdentityState =
  | { status: "anonymous" }
  | { status: "verified"; subject_ref: string; verified_at: string; expires_at: string }
  | { status: "failed"; failure_code: string };

type KnowledgeEvidence = {
  policy_id: string;
  version: string;
  section_ref: string;
  source_hash: string;
  effective_from: string;
  effective_to?: string;
};

type EligibilityDecision = {
  kind: "return" | "cancellation";
  eligible: boolean;
  reason_code: string;
  rule_version: string;
  evidence_refs: string[];
  decided_at: string;
};

type ActionProposal = {
  proposal_id: string;
  action: "create_return" | "cancel_order";
  target_ref: string;
  parameter_hash: string;
  rule_version: string;
  created_at: string;
  expires_at: string;
};

type Approval = {
  proposal_id: string;
  actor_subject_ref: string;
  decision: "approved" | "declined";
  recorded_at: string;
  channel: "web_click";
};

type PendingWrite = {
  operation: "create_ticket" | "create_return" | "cancel_order";
  idempotency_key: string;
  status: "not_started" | "in_flight" | "unknown" | "verified" | "rejected";
  external_ref?: string;
};

type WorkflowState = {
  case_id: string;
  correlation_id: string;
  stage: Stage;
  identity: IdentityState;
  interpretation?: DerivedInterpretation;
  confirmed_intent?: Intent;
  order_ref?: string;
  item_refs: string[];
  knowledge_evidence: KnowledgeEvidence[];
  eligibility?: EligibilityDecision;
  proposal?: ActionProposal;
  approval?: Approval;
  pending_write?: PendingWrite;
  escalation_reason?: string;
  llm_budget: { calls: number; input_tokens: number; output_tokens: number; cost_usd: number };
  version: number;
  created_at: string;
  updated_at: string;
  expires_at: string;
};
```

### State invariants

- LLM output cannot set identity, ownership, eligibility, approval, idempotency, verified success or terminal state.
- Protected tool context is added only after deterministic authentication and authorization.
- Candidate identifiers must resolve within the verified customer's allowlisted records before use.
- `EXECUTING` requires fresh authoritative state, deterministic eligibility and approval bound to the exact proposal hash.
- An `unknown` write moves only to `RECONCILING`; no model can request a blind retry.
- Mandatory escalation rules override model confidence and requested tool use.
- Terminal states reject subsequent writes; a new request creates a new case.

## Proposed database model

| Table | Minimum fields | Purpose and minimisation |
|---|---|---|
| `cases` | `case_id`, stage, confirmed intent, identity status, pseudonymous subject/order refs, timestamps, version, expiry | Authoritative workflow recovery; no transcript |
| `turn_buffer` | `turn_id`, `case_id`, role, redacted text or structured selection, timestamp, expiry | Short-lived recent context only; encrypted and TTL-enforced |
| `model_runs` | run ID, case ID, purpose, model/config version, prompt-template version, input/output hashes, token/cost/latency, validation result | Reproducibility without storing full prompts or outputs |
| `derived_interpretations` | run ID, candidate fields, confidence, risk/ambiguity codes, source turn refs, expiry | Clearly non-authoritative and replaceable |
| `knowledge_evidence` | case ID, policy/version/section/hash, effective dates | Grounding provenance; avoid passage copies unless active response requires one |
| `eligibility_decisions` | decision ID, case ID, result, reason code, rule version, evidence refs, timestamp | Deterministic authority |
| `action_proposals` | proposal ID, case ID, action, target ref, parameter hash, expiry, status | Immutable approval target |
| `approvals` | proposal ID, pseudonymous actor ref, decision, timestamp, channel | Explicit confirmation evidence |
| `operation_ledger` | idempotency key, case ID, operation, request hash, status, external ref, timestamps | Duplicate prevention/reconciliation |
| `escalations` | case ID, reason code, ticket ref/status, redacted structured summary, evidence refs | Minimal handoff; no automatic transcript attachment |
| `audit_events` | event ID, case/correlation IDs, actor type, event type, redacted metadata, source versions, timestamp | Append-only control evidence |
| `retained_case_summaries` | summary ID, case ID, structured facts, provenance, purpose, status, expiry, correction/deletion metadata | Exceptional long-term memory only |

Customer, order, shipment, return and cancellation systems remain authoritative. Vector indexes contain approved public knowledge only for the MVP; customer conversations and case summaries are not embedded.

## Context construction for each LLM call

Use a deterministic context builder in this order:

1. Fixed system rules and allowed output schema.
2. Current workflow stage and allowed capabilities.
3. Minimum recent redacted turns needed for the immediate task.
4. Confirmed structured facts relevant to that task.
5. Approved policy passages with version/hash, when needed.
6. Minimal protected tool result only after authorization.

Exclude unrelated prior cases, raw credentials, full profiles, addresses, payment data, audit logs, internal secrets and prohibited capabilities. Tool results and retrieved text are untrusted data, never instructions.

## Knowledge model

- Only approved FAQs/policies with owner, status, version, effective dates and content hash enter retrieval.
- Retrieval is filtered by topic and applicable date; conflicting or missing sources force handoff.
- Policy text supports explanations. Deterministic rule versions—not generated interpretations—decide eligibility.
- Retrieval records source metadata and passage reference. Do not retain entire retrieved documents in the case.
- Knowledge-index rebuilds preserve version traceability and remove expired/deleted content.

## Memory read rules

1. Default model access is the current case only.
2. Read at most the minimum recent turns required; prefer a structured case snapshot over chat history.
3. Public knowledge may be retrieved without authentication.
4. Protected facts require valid session binding, ownership checks and field-level allowlists outside the model.
5. Resume requires renewed authentication and refresh of mutable order state; prior model interpretations are revalidated.
6. Long-term summaries are not injected by default. Read them only for a declared repeat/unresolved-case purpose, same verified subject, unexpired record and field-level need.
7. Disputed, expired, deleted or provenance-free facts are excluded.

## Memory write rules

### Active-case writes

- Keep raw text in `turn_buffer` only while needed for active context, debugging under approved sampling, or a pending handoff; enforce a short TTL.
- Persist structured interpretations with source-turn references and mark them non-authoritative.
- Persist hashes and metadata for model observability instead of full prompts/responses.
- Persist authoritative references, deterministic decisions, approvals and verified outcomes.
- Redact and validate any model-written escalation summary before ticket creation.
- Do not embed customer turns, tool responses or case summaries in a vector database.

### Long-term write criteria

A fact may enter `retained_case_summaries` only when all are true:

1. A documented purpose exists: unresolved handoff, legally required audit evidence, or duplicate/repeat-case prevention.
2. The fact is necessary, structured and narrowly scoped.
3. Provenance identifies the source and observation time.
4. Verification status is explicit; model inference alone is insufficient.
5. An approved expiry is set.
6. The workflow records who/what wrote it and why.
7. The fact is available for user access, correction and deletion.

```ts
type RetainedFact = {
  fact_code: string;
  value: string | number | boolean;
  provenance: {
    source_type: "customer" | "authoritative_system" | "human_agent";
    source_ref: string;
    observed_at: string;
  };
  purpose: "handoff" | "duplicate_prevention" | "required_audit";
  verification: "asserted" | "verified" | "disputed";
  expires_at: string;
  created_by: { actor_type: "workflow" | "human"; actor_ref: string };
  corrected_from?: string;
};
```

The LLM cannot independently create long-term memory. It may draft a candidate; deterministic validation and, for non-routine facts, human approval are required.

## Correction and deletion

- A verified customer can request access, correction or deletion of retained case facts.
- Corrections version the fact, preserve provenance and mark prior content `superseded` or `disputed`; future retrieval excludes it.
- Deletion removes or irreversibly de-identifies eligible summaries, turn buffers, caches and derived indexes, then creates a minimal deletion receipt.
- Legal/audit holds and jurisdictional retention require stakeholder decisions. Restricted records are access-blocked and excluded from model context.
- Cache and vector-index deletion must be propagated and testable, not limited to the primary database.

## Information that must never be remembered

- Passwords, session cookies, bearer/API tokens, one-time codes or security answers.
- Full card/bank data, CVV or raw payment credentials.
- Government IDs, biometric data or identity-document images.
- Complete conversations by default, including hidden/system prompts.
- Unnecessary names, emails, phone numbers, addresses or precise location.
- Another customer's data or any record that failed ownership verification.
- Raw authentication artifacts after the trusted verification result is produced.
- Model chain-of-thought, hidden reasoning or internal safety instructions.
- Unsupported fraud accusations, risk scores, emotional profiles or inferred sensitive traits.
- Prompt-injection content as an instruction or reusable memory.
- Raw tool payloads containing fields outside the allowlist.
- Personal preferences, shopping profiles, browsing history or marketing attributes; outside MVP scope.
- Refund/replacement promises, policy exceptions or unverified claims of completed actions.

## Retention defaults requiring stakeholder approval

- Turn buffer: active case plus a short operational TTL; immediate deletion where possible after structured extraction.
- Derived interpretation: active case plus crash-recovery window.
- Model-run metadata/hashes: shortest period needed for evaluation and incident review.
- Operational/audit evidence: shortest compliant period.
- Long-term case facts: purpose-specific expiry; never indefinite.
- Synthetic portfolio fixtures: resettable, labelled synthetic and isolated from real personal data.

## Option E-specific safety tests

- No protected or cross-case fact appears in prompts before authorization.
- Prompt injection cannot alter memory policy, tool permissions or workflow state.
- Deleted/corrected facts disappear from context, caches and indexes.
- Model-produced identifiers cannot bypass ownership checks.
- Summarisation does not convert allegations or uncertainty into verified facts.
- Token/cost fallback removes optional context without removing safety checks.
