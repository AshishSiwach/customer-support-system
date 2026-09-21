# Autonomy Matrix — Option E

## Purpose

This matrix assigns the lowest autonomy level that still provides meaningful value for every external action identified for Option E: a deterministic customer-support workflow containing bounded LLM steps.

The assigned level applies to the **application as a whole**, not to the language model acting independently. Deterministic authentication, authorization, eligibility, confirmation, idempotency, escalation, verification, and audit controls remain authoritative at every level.

## Autonomy levels

- **Level 0 — Prohibited:** The application must not expose the capability.
- **Level 1 — Suggest only:** The application may recommend the action but cannot execute it.
- **Level 2 — Human approval:** The application prepares the action, but a human must approve execution.
- **Level 3 — Bounded autonomous:** The application may execute only for allowlisted targets and parameters.
- **Level 4 — Autonomous:** The application may execute without case-by-case approval.

For customer-owned actions, such as cancelling an order, explicit approval from the affected authenticated customer counts as human approval. It does not require approval from a support employee unless policy says otherwise.

## Global controls

Every non-prohibited external action must have:

- A deny-by-default capability registry.
- Typed inputs and outputs with schema validation.
- Authentication and authorization enforced outside the LLM.
- Minimum-necessary data access.
- Correlation IDs across conversation, tool call, action and ticket.
- Immutable or append-only audit events.
- Timeouts and explicit failure states.
- No conversion of missing, stale or conflicting evidence into success.
- Redaction of secrets and unnecessary personal data.
- A safe fallback to clarification, refusal or human escalation.

State-changing actions additionally require:

- A deterministic policy decision.
- An exact target and parameter preview.
- A confirmation bound to that exact proposed action.
- A short confirmation expiry.
- An idempotency key.
- Post-action verification against the authoritative system.

## Summary

| Level | Actions |
|---|---:|
| Level 0 — Prohibited | P1–P9, M11 |
| Level 1 — Suggest only | T9 |
| Level 2 — Human approval | W1, W2, T8, M10 |
| Level 3 — Bounded autonomous | Most authenticated reads, workflow writes, communications, ticket operations and escalations |
| Level 4 — Autonomous | R1, R2, W7 |

No state-changing customer or order operation receives Level 4.

## 1. Read-only actions

| ID | Action | Level | Why this level is appropriate | Technical enforcement | Evidence required before increasing autonomy |
|---|---|---|---|---|---|
| R1 | Search/read public FAQs | **4** | Public, read-only, reversible in effect and low blast radius. Incorrect retrieval can still mislead, but it cannot expose customer records or change business state. | Read-only connector restricted to approved published FAQ sources; source/version metadata; grounding checks; response citations or trace links. | Already at the maximum level. Expansion to new sources requires content-owner approval and retrieval-quality evidence. |
| R2 | Search/read public policies | **4** | Read-only and normally non-sensitive. Autonomous access is necessary for timely policy answers, although only approved policy versions may be used. | Versioned allowlist of approved policies; effective-date filtering; read-only credentials; conflict detection; source recorded in trace. | Already at the maximum level. Expansion requires verified policy governance, freshness monitoring and conflict tests. |
| R3 | Check authenticated-session status | **3** | Sensitive identity context is involved, but automatic checking is necessary before every protected read or action. The target must be the active signed session only. | Server-side session validation; signed claims; expiry and audience checks; no model access to credentials; deny on uncertainty. | To move to Level 4 would require proof that the operation cannot be retargeted, complete tenant isolation, zero unauthorized-access findings and security approval. Level 3 is preferable even then. |
| R4 | Retrieve minimal customer profile | **3** | Read-only but privacy-sensitive. Meaningful support may require limited profile attributes, but only for the authenticated subject. | Field-level allowlist; session subject must equal customer ID; row-level authorization; response minimisation and redaction. | Demonstrated need for each additional field, privacy approval, zero cross-customer access in adversarial tests and complete access logs. |
| R5 | Search/list authenticated customer's orders | **3** | Required for order selection, but incorrect scoping could expose another customer's purchases. | Query constructed from trusted session identity, not model-supplied customer ID; row-level access control; result-count and field limits. | Zero cross-account retrieval in security tests, reliable order disambiguation and audited authorization decisions. |
| R6 | Retrieve order and item details | **3** | Necessary and read-only, but contains personal and commercial information and must be limited to a verified order owner. | Ownership check; allowlisted fields; order ID constrained to R5 results; read-only service role; traceable query. | Evidence of perfect ownership enforcement, stable schemas and no sensitive-field leakage under prompt-injection tests. |
| R7 | Retrieve shipment events | **3** | Necessary and verifiable, but may expose delivery location and timing. | Order-ownership check; allowlisted tracking fields; no unrestricted carrier lookup; read-only API token; freshness timestamp. | Zero cross-order disclosure, verified handling of conflicting events and privacy sign-off. |
| R8 | Retrieve return status/history | **3** | Needed to prevent duplicates and determine current state. Read-only but customer-sensitive. | Scope to authenticated customer's selected order/item; field allowlist; read-only access; result timestamp. | Demonstrated duplicate-return prevention, cross-account isolation and reliable status semantics. |
| R9 | Retrieve cancellation and fulfilment state | **3** | Required before cancellation; read-only but time-sensitive and capable of influencing a consequential action. | Scope to selected owned order; authoritative source; freshness limit; fail closed on inconsistent states. | Proven freshness, conflict handling, ownership isolation and no false cancellability in the evaluation set. |
| R10 | Check return eligibility inputs/result | **3** | Automatic checking provides core MVP value, but it must use approved rules and a fixed order/item target. | Deterministic rules engine; versioned rule set; typed facts; explanation trace; no LLM authority over the decision. | Stakeholder-approved rules, high-coverage boundary tests, zero critical false approvals and reliable policy-version selection. |
| R11 | Check cancellation eligibility inputs/result | **3** | As with returns, automatic checking is useful only inside a tightly bounded rule set and current fulfilment state. | Deterministic rules engine; atomic or freshly revalidated order state; versioned rules; fail closed. | Stakeholder approval, race-condition tests, zero false approvals and verified behaviour at fulfilment-state boundaries. |
| R12 | Read existing support-ticket history | **0** | It is optional for the MVP, highly privacy-sensitive and can expose unrelated historical complaints. A safer fallback is creating a new ticket with current-case context. | Do not register or expose a ticket-history read tool to the application. Human agents retain access through their normal interface. | A demonstrated continuity problem, approved minimum fields and time window, privacy review, per-customer authorization, redaction and measured benefit over current-case context. |
| R13 | Verify a created return | **3** | Read-only verification reduces greater harm by preventing false success messages. It must be tied to the just-created return and active case. | Verify using returned operation/return ID plus owned order ID; bounded retries; authoritative readback; timeout becomes escalation. | Consistent operation identifiers, reliable read-after-write semantics and tests covering timeout and eventual consistency. |
| R14 | Verify a cancellation | **3** | Verification is necessary before reporting success and has a narrow, known target. | Read back the exact order and cancellation event; compare expected transition; bounded retry; uncertainty triggers escalation. | Proven state-transition semantics, race-condition testing and reliable reconciliation after timeouts. |
| R15 | Verify a created escalation ticket | **3** | A ticket must exist before the customer is told a handoff succeeded. Target and expected result are narrow and auditable. | Read back exact ticket ID; confirm required fields and queue; do not expose unrelated ticket contents. | Reliable create/read correlation, no cross-ticket access and tests for partial ticket creation. |

## 2. Data-writing and business actions

| ID | Action | Level | Why this level is appropriate | Technical enforcement | Evidence required before increasing autonomy |
|---|---|---|---|---|---|
| W1 | Create a return request | **2** | This changes order operations and can create logistics cost. It is only partly reversible. The affected authenticated customer must approve the exact return. | Deterministic eligibility; exact order/item/quantity preview; customer confirmation token bound to action hash; expiry; idempotency key; post-action verification. | Before Level 3: stakeholder approval for no-confirmation cases, zero wrong-target actions, zero duplicates, proven reversibility, reliable verification and strong evidence that removing confirmation benefits users. Not recommended for this MVP. |
| W2 | Cancel an eligible order | **2** | Cancellation can stop fulfilment, lose a sale and become irreversible. Explicit approval from the authenticated customer is essential. | Fresh eligibility check; exact order preview; action-bound confirmation; short expiry; revalidate immediately before execution; idempotency and readback. | Before Level 3: zero wrong-order cancellations, tested fulfilment races, guaranteed reversal window, verified success semantics and explicit business/risk approval. Not recommended for this MVP. |
| W3 | Record explicit customer confirmation | **3** | Recording confirmation is necessary and low blast radius when it is bound to one active proposed action. | Server-generated action ID/hash; authenticated actor; append-only event; timestamp and expiry; reject vague or mismatched confirmations. | To expand scope: evidence that confirmations remain correctly bound across retries, reconnects, multiple open orders and adversarial language. |
| W4 | Create idempotency/action record | **3** | This safety write should occur automatically, but only for an allowlisted action and exact target. Incorrect records could block or duplicate actions. | Server-generated idempotency key; unique database constraint; atomic write; action type and target schema; retention policy. | Collision and concurrency tests, zero duplicate side effects, reliable recovery after partial failures and operational monitoring. |
| W5 | Record action outcome | **3** | Required for workflow consistency and audit, but must never be based on the model's assertion alone. | Outcome derived only from verified tool response/readback; append-only transition; authoritative identifiers; reconciliation job for uncertain states. | Demonstrated agreement with authoritative systems, tested eventual consistency and no false-success records. |
| W6 | Record tool calls and failures | **3** | Auditability is required, but unrestricted logging could leak personal data or secrets. | Schema allowlist; structured logs; redaction; no credentials or raw unnecessary payloads; access control and retention limits. | Privacy review, redaction tests, trace completeness metrics and zero secret leakage before capturing additional fields. |
| W7 | Record aggregate latency, usage and cost metrics | **4** | Automatic measurement is low risk and necessary for evaluation when metrics contain no message content or direct identifiers. | Aggregate/structured telemetry only; pseudonymous correlation; sampling controls; no raw prompts; integrity checks. | Already at maximum. Expanding metric content requires privacy review and demonstrated necessity. |
| W8 | Record evaluation outcomes | **3** | Automated recording is valuable, but labels, scores and traces must be restricted to the defined experiment and cannot automatically promote a release. | Versioned evaluation run; immutable inputs; typed metrics; provenance; separation between measurement and deployment approval. | Reproducible scores, reviewer agreement, protected test-set integrity and governance before results can influence deployment automatically. |

## 3. User communication actions

| ID | Action | Level | Why this level is appropriate | Technical enforcement | Evidence required before increasing autonomy |
|---|---|---|---|---|---|
| C1 | Send public FAQ or policy answer | **3** | Immediate answers are meaningful, but the domain and evidence must be bounded. Incorrect advice is externally visible and cannot truly be unsent. | Approved-topic classifier; grounded generation from R1/R2; source/version attached to trace; abstain or escalate on weak/conflicting evidence. | High grounded-answer accuracy, low unsupported-claim rate, policy-owner review and robust adversarial tests before broadening topics or removing grounding requirements. |
| C2 | Ask customer to sign in or verify identity | **3** | Necessary for protected workflows, but must never solicit credentials in chat or redirect outside an approved authentication path. | Fixed template; allowlisted first-party authentication route; no credential collection by model; anti-phishing UI treatment. | Security review, verified redirect integrity, usability evidence and zero credential collection in red-team tests. |
| C3 | Ask a clarification question | **3** | Clarification is safer than guessing and has low direct blast radius, but it can collect unnecessary sensitive information. | Allowlisted clarification purposes; prohibited-data filter; maximum question count; no request for passwords/payment secrets; escalation fallback. | Evidence of improved task completion, low unnecessary-data collection and low conversation-loop rate before expanding question freedom. |
| C4 | Request missing order-selection information | **3** | Necessary when several owned orders match, but only minimal identifiers should be requested. | Present owned-order choices where possible; field allowlist; never request full payment data; validate against R5 results. | Zero cross-order selection, minimal-data review and strong disambiguation accuracy. |
| C5 | Present verified order status | **3** | Core value, but the content is sensitive and must be limited to the active authenticated customer and verified facts. | Authentication gate; server-side ownership check; response generated only from allowlisted fields; output-to-source consistency check. | Zero privacy violations and high factual consistency across a representative test set before expanding detail. |
| C6 | Present verified shipment status | **3** | Core value with location/privacy implications. Bounded presentation is safer than unrestricted disclosure. | Same controls as C5 plus suppression of unnecessary address/location data and shipment-freshness timestamp. | Zero location leakage, reliable carrier reconciliation and privacy approval. |
| C7 | Explain eligibility or ineligibility | **3** | Useful and reversible as communication, but it must reflect a deterministic decision and approved policy. | Explanation generated from structured rule result and cited policy clauses; model cannot alter decision; uncertainty becomes escalation. | High explanation fidelity, policy-owner review and no contradictions between explanation and rule result. |
| C8 | Present exact proposed return or cancellation | **3** | This is required before human approval and is not itself execution. It must be exact and unambiguous. | Deterministic action preview populated from authoritative state; order/item/quantity/action ID shown; no free-form mutation of parameters. | User-comprehension testing, zero preview/action mismatches and reliable handling of state changes after preview. |
| C9 | Request explicit confirmation | **3** | Necessary safety step; low risk when confirmation is action-bound and cannot be inferred from unrelated language. | Fixed confirmation UI or strict confirmation schema; display exact target; expiry; reject ambiguous replies; store W3. | Demonstrated correct binding, no accidental confirmations and successful tests across reconnects and repeated messages. |
| C10 | Report successful action | **3** | Provides core value, but false success would materially harm customers. It must occur only after verified completion. | Gate on R13/R14 and W5 verified state; fixed success fields; operation/reference ID; never infer success from timeout. | Zero false-success messages in injected-failure tests and reliable read-after-write verification. |
| C11 | Report failed or unavailable action | **3** | Necessary for transparency and recovery; bounded reporting is safer than silent retry. | Map typed tool failures to approved messages; do not expose internal secrets; uncertain outcome triggers escalation rather than retrying the write. | Complete error mapping, no secret leakage and customer-understanding tests. |
| C12 | Refuse unsupported or prohibited request | **3** | Required to preserve scope and safety. An incorrect refusal has limited reversibility but a human fallback exists. | Explicit capability policy; allowlisted refusal reasons; offer safe escalation; log reason and confidence. | Low false-refusal rate, complete prohibited-case coverage and human review of edge cases. |
| C13 | Inform customer of human handoff | **3** | Mandatory when escalation occurs, but should only claim success after ticket verification. | Gate on verified T1/R15 result; approved message; include expected next step without inventing an SLA. | Zero false handoff claims and reliable ticket-creation verification. |
| C14 | Provide escalation reference/ticket ID | **3** | Useful and verifiable, but the reference must belong to the active customer and case. | Use ticket ID returned by T1; case/session binding; do not expose internal-only or other-customer references. | Zero cross-ticket disclosure and verified case-reference integrity. |

## 4. Account and device operations

| ID | Action | Level | Why this level is appropriate | Technical enforcement | Evidence required before increasing autonomy |
|---|---|---|---|---|---|
| A1 | Validate existing website session | **3** | Required, privacy-sensitive and strictly scoped to the active session. The LLM must not decide validity. | Server-side authentication middleware; signed-token checks; expiry; session binding; deny by default. | Independent security testing, zero session-confusion failures and complete audit coverage. Level 4 is unnecessary. |
| A2 | Initiate/redirect to simulated sign-in | **3** | Useful for continuing protected support, but redirect targets and data collection must be tightly bounded. | Fixed first-party route; state/nonce validation; no model-generated URLs; no credentials in conversation. | Security review, redirect and CSRF tests, successful session binding and usability evidence. |
| A3 | Associate verified customer ID with active conversation | **3** | Necessary but high privacy impact if mismatched. It must rely only on trusted authentication claims. | Atomic server-side binding from signed subject; conversation ownership check; immutable binding event; re-authentication for changes. | Zero cross-customer binding under concurrency and adversarial tests; reliable reconnect behaviour. |

All device-control capabilities remain prohibited under P8 below.

## 5. Ticket operations

| ID | Action | Level | Why this level is appropriate | Technical enforcement | Evidence required before increasing autonomy |
|---|---|---|---|---|---|
| T1 | Create support ticket | **3** | Automatic ticket creation is the safe fallback for mandatory conditions. It is auditable and usually reversible by closing the ticket. | Allowlisted triggers and ticket schema; idempotency; authenticated case binding; ticket ID verification; rate limits. | Low duplicate and false-escalation rates, reliable ticket verification and support-owner approval before expanding triggers. |
| T2 | Attach structured handoff context | **3** | Necessary to avoid repetition, but only minimum relevant context should be added. | Fixed schema; field allowlist; redaction; provenance links to messages/tool results; maximum size. | Human-agent usefulness review, low omission rate and zero unnecessary-sensitive-data findings. |
| T3 | Record identity-verification status | **3** | Required for safe handoff, but must be copied from the authentication system rather than inferred. | Typed status from trusted middleware; timestamp and method; never store credentials; immutable source reference. | Agreement with authentication logs and privacy/security approval. |
| T4 | Record customer intent and affected order | **3** | Necessary for routing and continuity, but target errors could mislead humans. | Intent marked model-derived; order ID selected only from authenticated orders; provenance and confidence; human can correct. | High intent accuracy, zero wrong-customer order IDs and human correction-rate evidence. |
| T5 | Record evidence and attempted actions | **3** | Necessary to prevent repeated actions; can be automated when copied from authoritative traces. | Structured evidence references; verified operation IDs; no model-authored claim accepted as tool evidence; redaction. | High completeness, no unsupported claims and human-agent review. |
| T6 | Record escalation reason | **3** | Required and bounded by an explicit set of mandatory reasons. It does not itself determine specialist treatment. | Enumerated reason codes; supporting evidence reference; model cannot invent new privileged categories; audit event. | High reason accuracy and low correction rate across labelled escalation cases. |
| T7 | Route ticket to general human-support queue | **3** | General handoff is the safest fallback and has limited blast radius. | One allowlisted general queue; verified ticket; idempotent routing; queue receipt confirmation. | Reliable queue delivery, low lost-ticket rate and support-owner sign-off. |
| T8 | Route high-risk case to specialist queue | **2** | Fraud, legal and safety labels can be highly sensitive and stigmatizing; exact routing policy is unresolved. A human should approve specialist routing. | Application prepares suggested queue and evidence; general support/risk triage approves; no automatic adverse account action. | Stakeholder-defined taxonomy, calibrated high-risk classifier, bias testing, privacy/legal review, low false-positive rate and monitored overrides before Level 3. |
| T9 | Assign ticket category or priority | **1** | Priority affects service order and exact SLA rules are unresolved. Suggesting a category/priority adds value without silently changing queue treatment. | Store recommendation separately from authoritative fields; show rationale/evidence; human agent accepts or edits it. | Approved taxonomy and SLAs, high agreement with trained reviewers, acceptable subgroup error rates and auditability before Level 2 or 3. |

## 6. Memory and workflow-state operations

| ID | Action | Level | Why this level is appropriate | Technical enforcement | Evidence required before increasing autonomy |
|---|---|---|---|---|---|
| M1 | Create active conversation record | **3** | Required for multi-turn support and bounded to one session/customer. | Server-generated conversation ID; authenticated ownership where applicable; tenant isolation; expiration metadata. | Concurrency and isolation tests, no cross-session access and approved retention defaults. |
| M2 | Append customer and system messages | **3** | Required for conversation continuity, but transcripts contain sensitive data and should be append-only. | Conversation ownership; append-only event model; redaction; size/rate limits; encryption and access controls. | Transcript integrity, privacy review, redaction effectiveness and retention compliance. |
| M3 | Retrieve current conversation context | **3** | Necessary for multi-turn interaction but dangerous if context crosses sessions or customers. | Retrieve by trusted conversation/session ID; ownership check; bounded context window; no global memory search. | Zero cross-session retrieval, stable reconnect behaviour and context-selection evaluation. |
| M4 | Store extracted intent and entities | **3** | Useful for routing, but values are non-authoritative and must be constrained to the active case. | Typed schema; confidence/provenance; entity validation against authoritative data; label as model-derived. | High extraction accuracy, no sensitive over-collection and reliable correction when customer clarifies. |
| M5 | Store current workflow stage | **3** | Essential for enforcing order of operations. Only valid state transitions should be accepted. | Deterministic state machine; transition allowlist; compare-and-set/versioning; append-only history. | State-machine property tests, concurrency tests and zero skipped safety gates. |
| M6 | Store pending proposed action | **3** | Necessary for confirmation binding, but cannot itself authorize execution. | Typed immutable proposal; exact target/parameters; expiry; eligibility-version reference; unique action ID. | Zero proposal/execution mismatches and reliable invalidation when underlying state changes. |
| M7 | Store confirmation and action binding | **3** | Required safety evidence and bounded to one proposal. | Authenticated actor; action hash; timestamp; expiry; append-only event; reject reuse. | No stale, replayed or ambiguous confirmations across adversarial and reconnect tests. |
| M8 | Store tool results used as evidence | **3** | Needed for traceability, but raw results can contain unnecessary sensitive information. | Typed result subset; provenance; timestamp; redaction; integrity hash; access controls. | Trace completeness, source/result agreement and privacy review before retaining broader payloads. |
| M9 | Store escalation summary | **3** | A structured summary is necessary for meaningful handoff and is reviewable by the receiving human. | Summary marked AI-generated; required fields; evidence links; sensitive-field filter; raw facts remain available for comparison. | High factual consistency, human usefulness scores, low correction rate and zero fabricated evidence. |
| M10 | Expire/delete conversation memory | **2** | Deletion is destructive and the retention period is unresolved. Until policy is approved, a human or administrator should approve deletion batches/rules. | Retention job in dry-run mode; approval workflow; legal-hold check; deletion manifest; separate protected audit retention. | Approved retention schedule, legal/privacy sign-off, restore or recovery design where required, dry-run accuracy and deletion verification before Level 3. |
| M11 | Store long-term customer preferences | **0** | The brief explicitly states that persistent preference memory is unnecessary. It adds profiling, staleness and deletion risk without MVP value. | Do not define a preference-memory store or tool; prohibit cross-case semantic memory writes; tests ensure session data is not reused as preference. | A validated user need, explicit consent model, retention/deletion controls, privacy impact assessment and measurable benefit. Not eligible for the current MVP. |

## 7. Escalation actions

| ID | Action | Level | Why this level is appropriate | Technical enforcement | Evidence required before increasing autonomy |
|---|---|---|---|---|---|
| E1 | Escalate when customer requests a human | **3** | This is an explicit customer request and escalation is the safer fallback. Human approval before escalation would defeat the purpose. | Deterministic phrase/UI trigger; create one idempotent ticket; verify ticket; stop unsupported processing. | High detection recall, low duplicate-ticket rate and verified queue delivery. |
| E2 | Escalate after failed identity verification | **3** | Continuing would create privacy risk. Automatic escalation is safer than exposing protected data or repeatedly challenging the customer. | Authentication failure event from trusted service; no protected data in ticket; bounded retry count; general queue only. | Zero protected-data leakage and support review of verification-failure handoffs. |
| E3 | Escalate refund, replacement or policy-exception request | **3** | The requested action is outside application authority; creating a human review is the safe and necessary fallback. | Explicit out-of-scope intent codes; ticket schema; no financial or replacement tool registered; customer informed without promising outcome. | High detection recall, low false containment and complete absence of unauthorized financial actions. |
| E4 | Escalate fraud, threats, legal or safety concerns | **3** | Immediate handoff is safer than autonomous resolution. This level authorizes creation of a neutral high-risk-review ticket, not an accusation or adverse action. | Conservative trigger rules/classifier; neutral reason code; minimum data; general/high-risk review queue approved in advance; no account sanctions. | Recall-focused evaluation, bias and privacy review, human override analysis and incident simulations. Specialist routing remains T8 Level 2. |
| E5 | Escalate when required tool fails | **3** | Tool failure removes the evidence needed for safe completion. Automatic handoff avoids guessing or uncontrolled retries. | Typed failure states; bounded read retries; no automatic retry of uncertain writes; include correlation ID and tool error class. | Complete failure detection, no false-success paths and reliable ticket creation under dependency outages. |
| E6 | Escalate missing, ambiguous or conflicting evidence | **3** | Human review is the safest fallback and has lower potential harm than selecting one unsupported interpretation. | Conflict/missing-data checks outside the LLM; evidence attached; no action execution while state is uncertain. | High recall on conflict test cases, acceptable unnecessary-escalation rate and human confirmation that supplied evidence is useful. |
| E7 | Escalate damaged, incorrect or missing-item report | **3** | The MVP cannot issue refunds or replacements, so automatic ticket creation is necessary and reversible. | Allowlisted issue types; structured facts; no promise of outcome; idempotent ticket creation; evidence requirements bounded. | High routing accuracy, low duplicate rate and support-agent review of summaries. |
| E8 | Transfer structured handoff summary | **3** | Necessary to prevent repetition. It is bounded to the verified case and remains reviewable by the receiving human. | Required schema; provenance links; AI-generated label; redaction; ticket ownership and field limits. | High factual consistency and completeness, low human correction rate and no sensitive-data over-sharing. |
| E9 | Stop further state-changing processing after escalation | **3** | This is a protective workflow action. It must occur automatically for allowlisted terminal escalation states. | Deterministic terminal state; capability revocation for the run; cancellation of pending writes; transition audit; human-only resume. | State-machine tests showing zero post-escalation writes, safe recovery and correct human-resume behaviour. |

## 8. Explicitly prohibited capabilities

| ID | Capability | Level | Why this level is appropriate | Technical enforcement | Evidence required before increasing autonomy |
|---|---|---|---|---|---|
| P1 | Issue refund | **0** | Financial impact, fraud risk and policy authority are outside the MVP. A safer human-review fallback exists. | No refund tool or credential available; network/service denylist; refund requests trigger E3. | Approved refund policy, financial controls, fraud review, limits, reconciliation, audit, reversal process and staged human-approved pilot. |
| P2 | Create replacement shipment | **0** | Creates fulfilment and inventory obligations and is explicitly excluded. | No replacement tool; requests trigger E3; tests assert capability absence. | Approved eligibility rules, inventory controls, address verification, fraud controls, idempotency and human-approved pilot. |
| P3 | Grant policy exception | **0** | Exceptions require discretionary business authority and may create unfair or inconsistent treatment. | Rules engine cannot be overridden by model; exception path only creates ticket; immutable decision source. | Formal delegated-authority policy, fairness review, monetary limits, monitoring and human approval process. |
| P4 | Change address or customer/account details | **0** | High privacy and fulfilment-fraud risk; explicitly excluded. | No write scopes for profile/order-address services; block corresponding tool registration; escalate request. | Strong re-authentication, fraud controls, change notification, rollback, audit and stakeholder approval. |
| P5 | Process or modify payment information | **0** | High financial and regulatory exposure and outside scope. | No payment credentials/tools; prohibit collection of card data; redact detected payment secrets; escalate. | Separate compliant payment architecture, legal/security approval and dedicated threat model. Not part of this project. |
| P6 | Product recommendation or shopping assistance | **0** | Outside the defined post-purchase scope and unnecessary for evaluating the selected workflows. | Scope classifier and refusal/redirect; no catalogue recommendation tools. | Validated customer need, catalogue data, recommendation-quality and fairness evaluation, and a revised project scope. |
| P7 | Send email, SMS, voice or social messages | **0** | Only website live chat is approved; additional channels create identity, consent and delivery risks. | No channel credentials/connectors; outbound network allowlist; channel requests remain in chat or escalate. | Channel-specific identity, consent, retention, delivery and abuse controls plus stakeholder-approved expansion. |
| P8 | Control or access customer device | **0** | No device-support requirement exists; blast radius and privacy risk are unacceptable. | No device permissions, remote-control tools or file/media access; client exposes chat only. | A separate validated use case and security architecture. It should not be added to this MVP. |
| P9 | Make adverse account or fraud decisions | **0** | The system may escalate suspicious cases but must not accuse, block, suspend or penalize a customer. | No account-sanction tools; neutral escalation labels; human specialist owns decisions; audit tests. | Formal decision policy, due-process safeguards, bias evaluation, legal approval and human oversight. Not suitable for this MVP. |

## Final autonomy posture

The recommended posture is:

- **Level 4 only for low-risk public reads and aggregate telemetry.**
- **Level 3 for tightly scoped reads, communications, workflow state, audit writes, ticket creation and safe escalation.**
- **Level 2 for returns, cancellations, specialist routing and destructive retention operations.**
- **Level 1 for unresolved priority/category recommendations.**
- **Level 0 for financial, discretionary, profile-changing, multichannel, device-control, long-term profiling and adverse-account capabilities.**

The application may be conversationally flexible, but it remains operationally constrained. Increasing autonomy is justified only by measured reliability, verified target selection, strong post-action verification, approved policy, complete auditability and evidence that a less autonomous fallback materially harms user outcomes.
