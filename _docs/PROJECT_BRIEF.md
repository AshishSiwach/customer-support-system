# Project Brief: E-commerce Customer Support Agent

## Problem statement

Customers of a fictional e-commerce retailer currently rely on FAQs, email, and human-operated live chat for post-purchase support. Human agents manually interpret each request, verify the customer, inspect order and shipment information, apply return or cancellation policies, and either resolve or escalate the case.

This makes routine questions and policy-compliant requests unnecessarily dependent on human support. The proposed product is a website live-chat assistant that can answer general policy questions, investigate verified customers' orders, complete a small number of tightly controlled actions, and create a useful handoff when a case cannot be resolved safely.

The product must improve access to support without trading away correctness, privacy, policy compliance, or customer control.

## Target users

### Primary users

- Customers seeking help after placing an online order.
- Unauthenticated customers seeking general FAQ or policy information.
- Authenticated customers seeking order-specific information or an eligible return or cancellation.

### Secondary users

- Human customer-support agents receiving escalated cases.
- Support managers reviewing outcomes, failure patterns, and operating metrics.

Company size, industry subcategory, and geography are intentionally unspecified so they do not become fixed product requirements.

## Current workflow

1. A customer searches the retailer's FAQs or contacts support through email or live chat.
2. A human agent interprets the customer's problem.
3. For an order-specific request, the agent verifies the customer's identity.
4. The agent checks order, shipment, return, cancellation, and ticket information across internal systems.
5. The agent applies company policy and either answers the question, performs an action, requests further information, or escalates the case.
6. The agent records the outcome manually.

## Desired outcome

Deliver safe and correct autonomous resolution for routine, in-scope customer requests through website live chat. The system should resolve eligible cases without avoidable human involvement, while escalating promptly whenever identity, evidence, policy, tools, or risk conditions make autonomous resolution inappropriate.

Ticket reduction, response speed, customer satisfaction, and cost are important guardrails, but they must not be optimized at the expense of safe and correct outcomes.

## MVP use cases

1. **Answer public FAQ and policy questions**
   - Answer general questions about delivery, returns, and cancellations.
   - Do not reveal order or customer information before identity verification.

2. **Retrieve order and delivery status**
   - For an authenticated customer, retrieve the relevant order and shipment events.
   - Explain the current status without inventing missing information.

3. **Start an eligible return**
   - Check eligibility using the applicable policy and order facts.
   - Explain the proposed action and obtain explicit customer confirmation.
   - Start the return and report the recorded result.

4. **Cancel an eligible order**
   - Check whether the order is still cancellable.
   - Explain the proposed action and obtain explicit customer confirmation.
   - Cancel the order and report the recorded result.

5. **Triage and escalate unresolved or prohibited cases**
   - Gather relevant context for damaged, incorrect, or missing-item reports.
   - Create a structured support ticket when human intervention is required.
   - Include identity status, customer intent, order context, evidence gathered, attempted actions, and the escalation reason.

## Explicitly excluded use cases

- Issuing refunds or replacements autonomously.
- Granting policy exceptions.
- Changing delivery addresses, account details, or other customer data.
- Providing order-specific information to an unverified customer.
- Product discovery, recommendations, and general shopping assistance.
- Payment-failure and billing support.
- Email ingestion or asynchronous email conversations.
- Voice, telephone, social-media, or messaging-app support.
- Direct integration with a real retailer's production systems or use of real customer data.
- Fully autonomous handling of fraud, threats, legal matters, or safety concerns.
- A general-purpose customer-service agent beyond the defined post-purchase scope.

## Available data and integrations

All data is synthetic and all operational integrations are realistic mock APIs.

### Data

- Customer profiles and simulated authenticated-session status.
- Orders, items, fulfilment states, and timestamps.
- Shipment and delivery-tracking events.
- Return and cancellation policies.
- Return and cancellation eligibility data.
- Support tickets and case history.
- FAQs and customer-service policy documents.

### Mock integrations

- Customer and order lookup.
- Shipment tracking.
- Return eligibility checking and return creation.
- Cancellation eligibility checking and order cancellation.
- Support-ticket creation and escalation.

Mocks should represent both successful responses and realistic failures, including timeouts, unavailable services, missing records, rejected actions, and conflicting evidence.

## Constraints

- Portfolio-quality MVP for a fictional e-commerce retailer.
- Three-to-four-week focused implementation window.
- Website live chat is the only customer channel in the MVP.
- Demonstration load: 100 conversations per day and up to five concurrent conversations.
- First useful streamed response within two seconds under normal load.
- Tool-assisted workflow completed within 15 seconds under normal load.
- Average variable cost of no more than USD 0.05 per conversation, excluding fixed hosting costs.
- Unauthenticated customers may access only public FAQs and policies.
- Order-specific data and actions require identity verification.
- A return or cancellation requires an eligibility check and explicit customer confirmation.
- State-changing actions must be auditable and safe against duplicate execution.
- The system must stop autonomous action and escalate when a mandatory handoff condition is detected.

## Risks

- **Privacy breach:** exposing customer or order data before successful verification or to the wrong customer.
- **Unauthorized action:** creating a return or cancellation without valid eligibility and explicit confirmation.
- **Duplicate action:** repeating a state-changing operation after a retry, timeout, or duplicated message.
- **Hallucination:** inventing an order status, shipment event, policy rule, or completed action.
- **Incorrect policy application:** using irrelevant, ambiguous, or outdated policy information.
- **Unsafe containment:** attempting to resolve a case that should have been escalated.
- **Tool failure:** treating a timeout, partial response, or conflicting result as successful evidence.
- **Prompt injection or manipulation:** allowing customer-provided text or retrieved content to override system rules or tool permissions.
- **Fraud and abuse:** assisting a suspicious request or failing to route it for specialist review.
- **Poor handoff:** escalating without enough context, forcing the customer to repeat the issue.
- **Latency or cost overrun:** exceeding the agreed response-time or per-conversation budget.
- **Misleading portfolio claims:** presenting synthetic evaluation results as evidence of real production performance.

## Assumptions

- The website can provide a simulated trusted session for authenticated customers.
- Policies are available in a clear, version-controlled form suitable for retrieval and rule definition.
- Return and cancellation eligibility can be determined from explicit business rules and order facts.
- Human support remains available for escalated cases, although the MVP will simulate the handoff by creating a ticket.
- Conversation context is retained for the active case; persistent personal-preference memory is unnecessary.
- The system records tool calls, decisions, confirmations, outcomes, latency, and cost for evaluation.
- Performance objectives will be measured using percentile-based latency, provisionally the 95th percentile, rather than averages alone.
- Synthetic test cases can adequately cover normal paths, edge cases, tool failures, and adversarial inputs for an MVP evaluation.

## Mandatory escalation conditions

The system must create a human-support handoff when:

- The customer explicitly requests a human.
- Identity verification fails.
- The requested outcome requires a refund, replacement, or policy exception.
- Fraud, threats, legal issues, or safety concerns appear.
- A required tool fails.
- Available evidence is missing, ambiguous, or conflicting.

## Open questions

The following require stakeholder validation in a real implementation:

- What authentication method and verification strength are required for each action?
- What exact rules define return and cancellation eligibility?
- Which policy version applies when policies change during an active order or case?
- What are the retailer's operating countries, legal obligations, and data-residency requirements?
- What response-time target applies to each human escalation priority?
- Who owns and approves FAQ and policy updates?
- How long should conversations, tool traces, and audit records be retained?
- Which languages and accessibility requirements must be supported?
- What customer evidence is required for damaged, incorrect, or missing items?
- How should suspicious patterns be routed to fraud specialists?
- What minimum evaluation thresholds should stakeholders approve before release?
- How should customer satisfaction and repeat-contact rates be measured in a synthetic portfolio setting?

## Measurable success criteria

The figures below are proposed MVP acceptance thresholds and would require stakeholder approval before a real deployment.

### Primary quality and safety criteria

- At least **90% safe and correct resolution** across eligible, in-scope evaluation cases.
- **100% recall for mandatory escalation cases** in the release-blocking evaluation set.
- **Zero critical privacy, authorization, or policy violations** in the release-blocking evaluation set.
- **100% of return and cancellation executions** preceded by a recorded eligibility result and explicit customer confirmation.
- **Zero duplicate state-changing actions** during retry, timeout, and repeated-message tests.
- At least **95% correct tool selection and valid tool arguments** across the tool-use evaluation set.
- Every escalation ticket contains the required identity, intent, evidence, action, and escalation-reason fields.

### Performance and cost criteria

- Support **five concurrent conversations** without errors attributable to concurrency.
- Process a simulated workload equivalent to **100 conversations per day**.
- First useful streamed response within **two seconds** under normal load.
- Tool-assisted workflow completed within **15 seconds** under normal load.
- Average model, embedding, and retrieval cost at or below **USD 0.05 per conversation**.

### Secondary diagnostic metrics

- Eligible-case autonomous-resolution rate.
- Correct escalation rate and unnecessary-escalation rate.
- Grounded-answer accuracy and unsupported-claim rate.
- Tool failure and recovery rate.
- End-to-end latency by use case.
- Cost by use case and conversation length.
- Simulated repeat-contact rate after a purported resolution.

## Simpler non-agent baseline

Build a deterministic self-service support flow as the comparison baseline:

1. Present buttons for **Track order**, **Return an item**, **Cancel an order**, **Report an item problem**, and **Read policies**.
2. Use keyword or conventional search for FAQ and policy pages.
3. Require a simulated authenticated session before showing order-specific options.
4. Retrieve order and shipment data using fixed request handlers.
5. Apply coded eligibility rules for returns and cancellations.
6. Show the proposed action and require an explicit confirmation click.
7. Execute the selected action through the corresponding mock API.
8. Create a structured support ticket for unsupported requests, rule failures, unavailable services, or customer-requested handoff.

This baseline uses menus, forms, fixed rules, and templates rather than open-ended language-model planning or tool selection. It should be evaluated on the same cases, latency, cost, and safety measures. The agentic approach is justified only if it handles natural-language variation and multi-turn ambiguity materially better without reducing safety or reliability.
