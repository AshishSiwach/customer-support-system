# 001 Project Pattern

## Decision

Develop the customer-support project in two comparable stages:

1. **Option A — deterministic baseline**
2. **Option E — deterministic workflow containing bounded LLM steps**

Option A is the initial product baseline. Option E is the portfolio-grade AI extension. The LLM version must demonstrate measurable value over the baseline; it is not assumed to be better merely because it is more agentic.

## Why this pattern was selected

The core MVP workflows are predictable and policy-bound:

- Retrieve order and shipment information.
- Check return eligibility.
- Check cancellation eligibility.
- Obtain explicit customer confirmation.
- Start an eligible return or cancellation.
- Create and escalate support tickets.

These requirements can be satisfied with deterministic software. Authentication, authorization, eligibility, confirmation, idempotency, state-changing actions, audit logging, and mandatory escalation should therefore be implemented as reliable software controls rather than delegated to an LLM.

The potential value of an LLM is narrower:

- Understand varied natural-language requests.
- Extract intents and relevant entities.
- Ask useful clarification questions.
- Find and explain relevant policy information.
- Convert verified system results into customer-friendly responses.
- Produce structured summaries for human agents.

## Phase 1: deterministic baseline

Build a guided self-service support system without an LLM.

### Interaction pattern

- Present explicit options such as **Track order**, **Return an item**, **Cancel an order**, **Report an item problem**, and **Read policies**.
- Use forms and buttons to collect required information.
- Use keyword or conventional search for FAQ and policy content.
- Require an authenticated session for order-specific information and actions.
- Show the exact proposed action before requesting confirmation.

### Deterministic controls

- Authentication and authorization checks.
- Return and cancellation eligibility rules.
- Mandatory escalation rules.
- Action confirmation state.
- Idempotency protection for state-changing operations.
- Tool timeouts, retries, and error handling.
- Structured ticket creation.
- Audit events, latency measurement, and outcome logging.

### Purpose of the baseline

- Prove that the business workflows function correctly.
- Establish the minimum safety and reliability level.
- Produce a reusable evaluation dataset.
- Measure task completion, escalation, latency, and operating cost without an LLM.
- Identify where guided flows or keyword search genuinely fail customers.

The baseline is not throwaway code. Its business rules, mock APIs, action services, audit layer, tests, and evaluation cases should be reused by the LLM-enhanced version.

## Phase 2: deterministic workflow with bounded LLM steps

Add LLM capabilities around the Phase 1 controls while preserving deterministic ownership of consequential decisions and actions.

### LLM responsibilities

- Classify free-text customer intent.
- Extract non-authoritative entities from the conversation.
- Detect ambiguity and propose clarification questions.
- Retrieve and explain relevant public policy content.
- Render verified order and shipment data in natural language.
- Draft structured escalation summaries.

### Responsibilities that remain deterministic

- Deciding whether the customer is authenticated.
- Controlling which customer and order records are accessible.
- Calculating return and cancellation eligibility.
- Enforcing mandatory escalation conditions.
- Recording and validating explicit confirmation.
- Executing returns and cancellations.
- Preventing duplicate actions.
- Verifying action outcomes.
- Maintaining the authoritative workflow state and audit record.

### Intended workflow

1. Receive the customer's free-text message.
2. Check for explicit human-handoff and high-risk conditions.
3. Use the LLM to interpret the request where necessary.
4. Route into an explicit supported workflow.
5. Apply authentication and authorization gates.
6. Retrieve permitted order, shipment, and policy evidence.
7. Apply deterministic business rules.
8. Use the LLM to explain the verified result or ask a clarification question.
9. For an eligible action, display the exact proposed action and obtain explicit confirmation.
10. Execute one idempotent action through deterministic application code.
11. Verify the recorded outcome.
12. Respond to the customer or create a structured escalation ticket.

## Baseline-versus-LLM comparison

Both versions must be evaluated using the same test cases, mock-system states, failure injections, and scoring rules.

### Primary comparison measures

- Safe and correct resolution rate for eligible cases.
- Recall for mandatory escalation cases.
- Critical privacy, authorization, and policy violations.
- Customer task-completion rate.
- Correct intent and workflow routing.
- Unsupported-claim or hallucination rate.
- Unnecessary-escalation rate.
- Repeat-contact rate after an apparent resolution.
- End-to-end latency.
- Variable cost per conversation.

### Promotion rule

Option E should be considered justified only if it:

- Produces a material improvement in handling realistic natural-language variation or multi-turn ambiguity.
- Meets or exceeds the baseline's safety and policy-compliance results.
- Preserves mandatory escalation recall.
- Remains within the agreed latency and cost limits.
- Does not introduce unacceptable nondeterminism or operational complexity.

If the LLM version does not meet these conditions, Option A remains the preferred product architecture.

## Explicitly rejected pattern

A supervisor-worker or multi-agent system is not justified for this MVP. The supported workflows are short, closely related, and dependent on the same customer and order state. There is no demonstrated need for independent specialist agents, parallel investigations, recursive delegation, or cross-agent negotiation.

Multi-agent architecture should be reconsidered only if future evidence shows several genuinely independent domains, different permission boundaries, long-running parallel work, or a measured improvement that outweighs additional latency, cost, coordination failures, and evaluation burden.

## Portfolio interpretation

The portfolio story is not simply that an AI agent was built. It is that architectural complexity was introduced only after establishing a reliable control:

> The deterministic baseline proved the workflows, safety invariants, and evaluation methodology. Bounded LLM steps were then added and measured against the same baseline to determine whether natural-language flexibility justified their additional cost and risk.

This demonstrates product judgement, controlled experimentation, tool integration, workflow orchestration, safety engineering, observability, and evaluation without treating greater autonomy as an objective by itself.
