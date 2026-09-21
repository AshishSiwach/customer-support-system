# Option E — Minimum Tool Catalogue

## Decision

Use five narrow tools with identical production and portfolio-simulation contracts.

| Tool | Model-visible? | Access | Autonomy |
|---|---:|---|---:|
| `search_support_policy` | Yes | Public read | Level 4 |
| `get_order_context` | Yes, only after workflow authorization | Protected read | Level 3 |
| `create_support_ticket` | No; workflow-controlled | State-changing | Level 3 |
| `create_approved_return` | No; workflow-controlled | State-changing | Level 2 |
| `cancel_approved_order` | No; workflow-controlled | State-changing | Level 2 |

The model may request a handoff or propose an action through structured output. It does not directly execute the three state-changing tools.

## Shared rules

- Deny by default; expose tools only in allowlisted workflow states.
- Validate all inputs and outputs against strict schemas.
- Derive customer identity from trusted session context, never model arguments.
- Log correlation ID, case ID, tool name, validated arguments, timing and result.
- Never convert timeout, missing data or conflicting evidence into success.
- Never blindly retry an uncertain state-changing request.
- Production and simulated implementations must pass the same contract tests.

---

## 1. `search_support_policy`

**Business purpose:** Find approved delivery, return or cancellation guidance for a public question.

**Model should use it when:** The interpreted intent is a general policy/FAQ question and no customer-specific data is required.

**Must not use it when:** Deciding eligibility; retrieving an order; answering from unapproved documents; resolving conflicting policies; searching for secrets or internal procedures.

### Typed input

```ts
type SearchSupportPolicyInput = {
  query: string; // 1..500 characters
  topic: "delivery" | "returns" | "cancellations";
  as_of?: string; // ISO-8601 date
  max_results?: number; // 1..5, default 3
};
```

### Typed output

```ts
type SearchSupportPolicyOutput =
  | {
      status: "found";
      results: Array<{
        policy_id: string;
        version: string;
        title: string;
        section: string;
        passage: string;
        effective_from: string;
        effective_to: string | null;
        source_hash: string;
      }>;
      conflicting_sources: boolean;
    }
  | { status: "not_found"; results: []; conflicting_sources: false };
```

**Authentication/authorization:** Public read credential; restricted to approved published policy sources.

**State:** Read-only.

**Idempotency:** Naturally idempotent for the same query, index and policy version.

**Timeout:** 2 seconds.

**Retry:** Once for timeout or transient service error. No retry for invalid input or `not_found`.

**Possible errors:** `INVALID_QUERY`, `TOPIC_NOT_ALLOWED`, `SOURCE_UNAVAILABLE`, `TIMEOUT`, `POLICY_CONFLICT`, `SCHEMA_ERROR`.

**Audit fields:** `correlation_id`, `case_id`, `query_hash`, `topic`, `as_of`, `policy_ids`, `versions`, `source_hashes`, `latency_ms`, `status`, `error_code`.

**Approval:** None.

**Independent verification:** Confirm returned IDs and hashes exist in the approved policy registry and that the effective dates cover `as_of`. A conflict prevents answering and triggers escalation.

**Production implementation:** Read-only search over the retailer's approved, versioned help centre and policy repository.

**Portfolio simulation:** Seeded policy documents with versions, effective dates and passage hashes. Include no-result, stale-version, timeout and conflicting-policy fixtures.

---

## 2. `get_order_context`

**Business purpose:** Retrieve the minimum verified order, item, shipment and existing return/cancellation state needed for the selected support case.

**Model should use it when:** The workflow has verified the session, bound the customer to the case and validated that `order_id` belongs to that customer.

**Must not use it when:** The customer is unauthenticated; the order was not selected from the owned-order set; searching by customer ID, email, address or arbitrary order reference; accessing ticket history.

### Typed input

```ts
type GetOrderContextInput = {
  order_id: string;
  view: "status" | "return_context" | "cancellation_context";
};
```

`customer_id` is injected by trusted workflow context and is not accepted from the model.

### Typed output

```ts
type GetOrderContextOutput = {
  status: "found";
  order: {
    order_id: string;
    order_status: string;
    placed_at: string;
    updated_at: string;
    version: string;
    items: Array<{
      item_id: string;
      name: string;
      quantity: number;
      fulfilment_status: string;
    }>;
  };
  shipment: {
    shipment_id: string | null;
    carrier_status: string | null;
    events: Array<{ event: string; occurred_at: string }>;
    updated_at: string | null;
  };
  existing_return: {
    return_id: string;
    item_id: string;
    status: string;
  } | null;
  cancellation: {
    cancellable_state: string;
    cancellation_id: string | null;
  };
};
```

**Authentication/authorization:** Valid session required. Server derives customer identity and enforces order ownership and field-level access.

**State:** Read-only.

**Idempotency:** Naturally idempotent, although the underlying order state may change; every result includes timestamps/version.

**Timeout:** 3 seconds.

**Retry:** Once for transient read failure. Do not retry authorization, not-found or conflicting-state errors.

**Possible errors:** `UNAUTHENTICATED`, `FORBIDDEN`, `ORDER_NOT_FOUND`, `OWNERSHIP_MISMATCH`, `SOURCE_UNAVAILABLE`, `TIMEOUT`, `CONFLICTING_STATE`, `SCHEMA_ERROR`.

**Audit fields:** `correlation_id`, `case_id`, trusted `customer_id`, `order_id`, `view`, source versions, fields returned, `latency_ms`, `status`, `error_code`.

**Approval:** None after deterministic authorization.

**Independent verification:** The adapter verifies ownership against the authoritative order system. Before a write, the workflow must call the authoritative source again and compare order version/state.

**Production implementation:** Read-only façade over order, fulfilment, shipment and returns systems, returning a minimal normalized view.

**Portfolio simulation:** Seeded customer/order database plus mock shipment and return records. Inject ownership mismatches, stale state, conflicting carrier events, timeouts and missing orders.

---

## 3. `create_support_ticket`

**Business purpose:** Create and verify a structured human-support handoff.

**Model should use it when:** Never directly. The model may emit `handoff_required` and draft a summary; the deterministic workflow validates the trigger and payload before calling the tool.

**Must not use it when:** Continuing automation is safe; creating specialist routing without human approval; assigning authoritative priority; attaching speculative or unverified facts.

### Typed input

```ts
type CreateSupportTicketInput = {
  case_id: string;
  reason:
    | "HUMAN_REQUESTED"
    | "IDENTITY_FAILED"
    | "REFUND_REPLACEMENT_EXCEPTION"
    | "HIGH_RISK_REVIEW"
    | "TOOL_FAILURE"
    | "CONFLICTING_EVIDENCE"
    | "ITEM_PROBLEM"
    | "UNRECOGNIZED_REQUEST";
  identity_status: "verified" | "unverified" | "failed";
  order_id?: string;
  summary: string;
  evidence_refs: string[];
  attempted_actions: string[];
  idempotency_key: string;
};
```

### Typed output

```ts
type CreateSupportTicketOutput =
  | {
      status: "created" | "already_exists";
      ticket_id: string;
      queue: "GENERAL_SUPPORT" | "HIGH_RISK_REVIEW";
      received_at: string;
      verified: true;
    }
  | {
      status: "unknown";
      ticket_id?: string;
      verified: false;
    };
```

**Authentication/authorization:** Internal service identity plus case-bound customer context. No customer credential is passed to the model.

**State:** State-changing.

**Idempotency:** Mandatory unique key derived from `case_id` and escalation event. Duplicate calls return the existing ticket.

**Timeout:** 5 seconds.

**Retry:** Once only after a definite pre-commit failure, using the same idempotency key. On timeout/unknown outcome, reconcile by key; never create with a new key.

**Possible errors:** `INVALID_REASON`, `INVALID_EVIDENCE`, `SENSITIVE_DATA_REJECTED`, `UNAUTHORIZED`, `DUPLICATE`, `TIMEOUT_UNKNOWN`, `QUEUE_UNAVAILABLE`, `VERIFICATION_FAILED`, `SCHEMA_ERROR`.

**Audit fields:** `correlation_id`, `case_id`, `reason`, `identity_status`, `order_id`, evidence references, summary hash, redaction result, idempotency key, `ticket_id`, queue, timing, verification result, error.

**Approval:** General-queue creation requires no approval. Specialist routing remains human-approved; priority/category remains suggestion-only.

**Independent verification:** Read the ticket back using the returned ID or idempotency key; confirm required fields and queue receipt before notifying the customer.

**Production implementation:** Adapter to the retailer's ticketing platform with least-privilege create/readback credentials and a fixed general queue.

**Portfolio simulation:** Mock ticket database with unique idempotency constraint and readback. Inject pre-commit failure, post-commit timeout, duplicate request, unavailable queue and incomplete-ticket cases.

---

## 4. `create_approved_return`

**Business purpose:** Execute exactly one eligible return after explicit customer approval.

**Model should use it when:** Never directly. The workflow calls it only after deterministic authentication, ownership, eligibility, proposal and confirmation checks.

**Must not use it when:** Eligibility is unknown; evidence conflicts; approval is missing/expired; parameters changed; a return already exists; refund/replacement is requested.

### Typed input

```ts
type CreateApprovedReturnInput = {
  proposal_id: string;
  approval_token: string;
  idempotency_key: string;
};
```

The server resolves customer, order, item, quantity and policy version from the immutable proposal. Raw target parameters are not accepted from the model.

### Typed output

```ts
type CreateApprovedReturnOutput =
  | {
      status: "completed" | "already_completed";
      return_id: string;
      order_id: string;
      item_id: string;
      quantity: number;
      verified: true;
    }
  | {
      status: "rejected";
      reason: string;
      verified: true;
    }
  | { status: "unknown"; verified: false };
```

**Authentication/authorization:** Valid current customer session; proposal customer must match session subject; server-side order ownership and action scope.

**State:** State-changing.

**Idempotency:** Mandatory unique key tied to proposal ID. Reuse returns the existing return rather than creating another.

**Timeout:** 6 seconds.

**Retry:** Never blindly retry after timeout. Reconcile by idempotency key/order/item. Retry only a definite pre-commit failure with the same key.

**Possible errors:** `UNAUTHENTICATED`, `FORBIDDEN`, `PROPOSAL_NOT_FOUND`, `PROPOSAL_EXPIRED`, `APPROVAL_INVALID`, `STATE_CHANGED`, `NO_LONGER_ELIGIBLE`, `RETURN_EXISTS`, `TIMEOUT_UNKNOWN`, `VERIFICATION_FAILED`, `SCHEMA_ERROR`.

**Audit fields:** `correlation_id`, `case_id`, trusted customer ID, proposal ID/hash, approval actor/time/token hash, policy/rule version, order/item/quantity, idempotency key, return ID, before/after state, timing and error.

**Approval:** Level 2—explicit approval from the authenticated affected customer.

**Independent verification:** Read back the return record and confirm customer, order, item, quantity and status match the approved proposal before reporting success.

**Production implementation:** Transactional adapter to the retailer's returns system, with proposal validation, idempotency ledger and authoritative readback.

**Portfolio simulation:** Mock returns table with deterministic eligibility fixtures, unique proposal/idempotency constraints and readback. Inject stale proposal, duplicate call, rejection, pre-commit failure and post-commit timeout.

---

## 5. `cancel_approved_order`

**Business purpose:** Cancel exactly one eligible order after explicit customer approval.

**Model should use it when:** Never directly. The workflow calls it only after fresh deterministic revalidation.

**Must not use it when:** Approval is absent/expired; fulfilment has advanced; eligibility is uncertain; order ownership is unverified; parameters differ from the proposal.

### Typed input

```ts
type CancelApprovedOrderInput = {
  proposal_id: string;
  approval_token: string;
  idempotency_key: string;
};
```

The server resolves the exact order and expected version from the immutable proposal.

### Typed output

```ts
type CancelApprovedOrderOutput =
  | {
      status: "completed" | "already_completed";
      cancellation_id: string;
      order_id: string;
      order_status: "cancelled";
      verified: true;
    }
  | {
      status: "rejected";
      reason: string;
      current_order_status: string;
      verified: true;
    }
  | { status: "unknown"; verified: false };
```

**Authentication/authorization:** Valid current customer session; proposal subject must match; server-side ownership and cancellation scope.

**State:** State-changing.

**Idempotency:** Mandatory unique key tied to proposal ID/order. Duplicate calls return the existing cancellation.

**Timeout:** 6 seconds.

**Retry:** Never blindly retry an unknown result. Reconcile by idempotency key and order state. Retry only definite pre-commit failure with the same key.

**Possible errors:** `UNAUTHENTICATED`, `FORBIDDEN`, `PROPOSAL_NOT_FOUND`, `PROPOSAL_EXPIRED`, `APPROVAL_INVALID`, `VERSION_CONFLICT`, `FULFILMENT_ADVANCED`, `NO_LONGER_ELIGIBLE`, `ALREADY_CANCELLED`, `TIMEOUT_UNKNOWN`, `VERIFICATION_FAILED`, `SCHEMA_ERROR`.

**Audit fields:** `correlation_id`, `case_id`, trusted customer ID, proposal ID/hash, approval actor/time/token hash, policy/rule version, order ID/version, idempotency key, cancellation ID, before/after state, timing and error.

**Approval:** Level 2—explicit approval from the authenticated affected customer.

**Independent verification:** Read the authoritative order record and cancellation event; confirm the order is cancelled and matches the approved proposal before reporting success.

**Production implementation:** Transactional adapter to order/fulfilment systems with optimistic concurrency, idempotency and authoritative readback.

**Portfolio simulation:** Mock order state machine with version checking and idempotency. Inject fulfilment race, duplicate call, rejection, pre-commit failure, post-commit timeout and verification mismatch.

---

## Operations that must remain ordinary deterministic code

Do not expose these as LLM-callable tools:

- Session validation and customer identity binding.
- Customer/order ownership enforcement.
- Capability and workflow-state gating.
- Return and cancellation eligibility calculation.
- Policy-version selection for eligibility.
- Proposal creation, hashing and expiry.
- Confirmation validation and action binding.
- Idempotency-key generation and ledger management.
- Retry classification and unknown-outcome reconciliation.
- Post-action verification.
- Mandatory-escalation decisions.
- Ticket priority/category authorization.
- Audit, redaction, retention and cost accounting.
- Conversation-state transitions and terminal-state enforcement.
- Sending messages; the model may draft text, but the workflow validates and delivers it.

## Capabilities absent from the catalogue

No tools exist for refunds, replacements, policy exceptions, address/profile changes, payments, product recommendations, email/SMS/voice, device control, long-term preference memory or adverse account/fraud decisions.
