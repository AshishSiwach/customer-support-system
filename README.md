# Customer Support System

A safety-first e-commerce customer support system designed around deterministic,
auditable workflows with bounded LLM assistance.

The project defines a portfolio-quality MVP for website live chat that can:

- answer grounded FAQ and policy questions;
- investigate authenticated customers' orders and shipments;
- create eligible returns or cancellations after explicit confirmation;
- prevent duplicate or unauthorized state-changing actions; and
- escalate ambiguous, sensitive, or failed cases with a structured handoff.

The current repository contains the product brief, baseline and AI-enhanced
architectures, implementation plan, autonomy matrix, state and memory designs,
technology choices, and minimum tool catalogue in [`_docs`](./_docs/).

## Architecture direction

The recommended implementation is a deterministic state machine with bounded
LLM interpretation and communication steps. Authentication, authorization,
eligibility, approvals, idempotency, execution, verification, and escalation
remain under deterministic control.

See [`_docs/PROJECT_BRIEF.md`](./_docs/PROJECT_BRIEF.md) and
[`_docs/ARCHITECTURE_Option_E.md`](./_docs/ARCHITECTURE_Option_E.md) for the
project scope and recommended architecture.

## Status

Design and implementation planning are complete. Application development is the
next phase.
