# Option A — State, Knowledge and Memory

## Decision

Option A uses a deterministic state machine. It has **workflow state and case records, not AI memory**. Only the minimum structured facts required to complete, resume, audit or escalate a case are retained. Complete chat transcripts are not long-term records by default.

## Data classes

| Class | Examples | Authority | Default lifetime |
|---|---|---|---|
| Ephemeral input | Current form values, current message, validation errors | Customer input until validated | Request or active session |
| Authoritative facts | Session subject, owned order state, shipment events, return/cancellation result | Source system | Referenced, not copied long-term |
| Workflow state | Current step, selected intent, identifiers, deadlines | Application | Active case + short recovery window |
| Derived decisions | Eligibility result, escalation reason | Versioned deterministic rules | Case/audit retention |
| Approval evidence | Proposal hash, customer confirmation, expiry | Application | Audit retention |
| Audit evidence | State transitions, tool outcomes, policy/rule versions | Append-only audit log | Stakeholder-defined retention |
| Long-term case memory | Minimal resolved/escalated case summary | Application | Only under explicit criteria below |

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
  | "INTENT_SELECTED"
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

type IdentityState =
  | { status: "anonymous" }
  | { status: "verified"; subject_ref: string; verified_at: string; expires_at: string }
  | { status: "failed"; failure_code: string };

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
  intent?: Intent;
  identity: IdentityState;
  order_ref?: string;
  item_refs: string[];
  policy_refs: Array<{ policy_id: string; version: string; source_hash: string }>;
  eligibility?: EligibilityDecision;
  proposal?: ActionProposal;
  approval?: Approval;
  pending_write?: PendingWrite;
  escalation_reason?: string;
  version: number;
  created_at: string;
  updated_at: string;
  expires_at: string;
};
```

### State invariants

- Protected order state is unavailable unless `identity.status === "verified"`.
- The session subject—not customer-entered identity—scopes all protected reads.
- `EXECUTING` requires a current eligible decision and unexpired approval for the exact proposal hash.
- One case has at most one active consequential proposal.
- An `unknown` write moves only to `RECONCILING`; it is never blindly retried.
- Terminal states reject further writes. A new request creates a new case.
- Missing, stale or conflicting evidence leads to `ESCALATING` or `FAILED_SAFE`.

## Proposed database model

| Table | Minimum fields | Purpose and minimisation |
|---|---|---|
| `cases` | `case_id`, `stage`, `intent`, `identity_status`, pseudonymous `subject_ref?`, `order_ref?`, timestamps, `version`, `expires_at` | Durable recovery state; no names, email, address or raw messages |
| `case_items` | `case_id`, `item_ref` | Only selected items; delete when no longer needed |
| `policy_evidence` | `case_id`, `policy_id`, `version`, `source_hash`, `section_ref` | Records which approved source governed the decision; avoid copying passages |
| `eligibility_decisions` | `decision_id`, `case_id`, `kind`, result, reason code, rule version, evidence refs, timestamp | Reproducible deterministic outcome |
| `action_proposals` | `proposal_id`, `case_id`, action, target ref, parameter hash, expiry, status | Immutable approval target; no free-text payload |
| `approvals` | `proposal_id`, pseudonymous actor ref, decision, timestamp, channel | Proof of explicit confirmation; never store authentication secrets |
| `operation_ledger` | `idempotency_key`, `case_id`, operation, request hash, status, external ref, timestamps | Duplicate prevention and unknown-outcome reconciliation |
| `escalations` | `case_id`, reason code, ticket ref, verified status, minimal structured summary, evidence refs | Human handoff without a transcript dump |
| `audit_events` | event ID, case/correlation IDs, actor type, event type, redacted metadata, source versions, timestamp | Append-only safety and evaluation trail |
| `retained_case_summaries` | summary ID, case ID, allowed fact codes, provenance refs, purpose, written by, expiry, correction status | Optional long-term memory under explicit criteria only |

Authoritative customer, order, shipment, return and cancellation records remain in their source systems. The application stores references and observed versions, not full replicas.

## Knowledge model

- Public FAQs and policies are approved, versioned records with effective dates, owner, status and content hash.
- Eligibility logic is deterministic, separately versioned code or decision tables.
- Policy retrieval may display text; operational eligibility always uses the applicable rule version.
- Draft, expired, conflicting or ownerless knowledge cannot support a final answer or action.
- Each answer or decision records source identifiers and versions, not an entire document copy.

## Memory read rules

1. Public knowledge may be read without authentication.
2. Case state may be read only by the active case and trusted service roles.
3. Protected source data requires a valid session, subject match and field allowlist.
4. A resumed case must revalidate session binding and refresh mutable order state.
5. Long-term case summaries are not loaded by default. They may be read only for a declared purpose, same verified subject, unresolved/repeat issue and unexpired retention.
6. Audit logs are never included in customer-facing context except through an approved, redacted support view.

## Memory write rules

### Ephemeral and operational writes

- Store the current message only long enough to validate and route it; discard it after extracting the selected structured fields unless needed for an active handoff.
- Persist state transitions, authoritative references, decisions, approvals and verified outcomes.
- Reject unnecessary fields at the API and database schema boundary.
- Redact free text before a ticket or audit write; prefer controlled reason codes.

### Long-term write criteria

A `retained_case_summary` may be written only when all are true:

1. A documented purpose exists: unresolved handoff, legally required audit evidence, or prevention of duplicate/repeated case work.
2. The fact is necessary, structured and relevant to that purpose.
3. Its provenance points to a customer statement or authoritative system observation.
4. Confidence/status is explicit; allegations and model inferences are not facts.
5. An expiry is assigned from an approved retention schedule.
6. The write is audited and visible through a customer/support correction process.

Required metadata:

```ts
type RetainedFact = {
  fact_code: string;
  value: string | number | boolean;
  provenance: { source_type: "customer" | "system" | "agent"; source_ref: string; observed_at: string };
  purpose: "handoff" | "duplicate_prevention" | "required_audit";
  status: "asserted" | "verified" | "disputed";
  expires_at: string;
  created_by: string;
  corrected_from?: string;
};
```

Complete conversations are never promoted to long-term memory by default.

## Correction and deletion

- Customers can request access, correction or deletion through a support flow linked to their verified identity.
- Corrections create a new version and mark the earlier fact `superseded` or `disputed`; immutable audit evidence records that a correction occurred without retaining unnecessary original content.
- Deletion removes or irreversibly de-identifies eligible case summaries and search indexes, then records a minimal deletion receipt.
- Legal/audit holds, jurisdictional rules and retention durations require stakeholder approval. When deletion is restricted, access is blocked and the reason/expiry is recorded.

## Information that must never be remembered

- Passwords, session cookies, access tokens, one-time codes, API keys or security answers.
- Full payment-card or bank details, CVV, or raw payment credentials.
- Government identifiers or identity-document images.
- Full customer conversations by default.
- Unnecessary names, email addresses, phone numbers, delivery addresses or precise location.
- Data belonging to another customer or records that failed ownership checks.
- Raw authentication evidence after verification.
- Unsupported suspicions, fraud labels, emotional profiling or inferred sensitive traits.
- Prompt-injection text, hidden instructions or retrieved secrets.
- Declined/expired action proposals beyond the minimal audit evidence required.
- Personal preferences, browsing history or marketing profiles; these are outside MVP scope.

## Retention defaults requiring stakeholder approval

- Active workflow state: until terminal state plus a short crash-recovery window.
- Raw current-turn text: active request only; active handoff text may survive until ticket verification.
- Operational/audit evidence: shortest period compatible with investigation and compliance.
- Retained case summaries: purpose-specific expiry; no indefinite records.
- Synthetic portfolio data: resettable and clearly separated from any real personal data.

