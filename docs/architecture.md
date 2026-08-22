# Architecture

## Overview

The **AI Support Operations Triage System** is an n8n-based support automation workflow designed to process incoming customer tickets, interpret unstructured messages with AI, apply deterministic business rules, route tickets to the correct support queue, generate safe response drafts, and persist the final operational record.

The system was intentionally designed around one core principle:

> **AI interprets. Deterministic systems control.**

The language model is responsible for understanding customer language.

The workflow itself remains responsible for validation, duplicate prevention, authorization boundaries, human-review decisions, routing, failure handling, and persistence.

This prevents the AI from becoming the final authority for sensitive business decisions.

---

# Architecture Blueprint

## Full Workflow

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                           INCOMING SUPPORT TICKET                            │
└───────────────────────────────────────┬──────────────────────────────────────┘
                                        │
                                        ▼
                                  ┌─────────────┐
                                  │   Webhook   │
                                  └──────┬──────┘
                                         │
                                         ▼
                           ┌──────────────────────────┐
                           │ Normalize Support Ticket │
                           └────────────┬─────────────┘
                                        │
                                        ▼
                           ┌─────────────────────────┐
                           │ Validate Support Ticket │
                           └───────────┬─────────────┘
                              VALID    │     INVALID
                              ┌────────┘        └──────────────┐
                              │                                │
                              ▼                                ▼
                  ┌──────────────────────┐          ┌─────────────────────────┐
                  │ Find Existing Ticket │          │ Prepare Rejected Ticket │
                  └──────────┬───────────┘          └────────────┬────────────┘
                             │                                    │
                             ▼                                    ▼
                  ┌────────────────────┐               ┌───────────────────────┐
                  │ Duplicate Ticket?  │               │ Append Rejected       │
                  └───────┬────────────┘               │ Tickets               │
                          │                            └───────────────────────┘
                    YES   │   NO
                 ┌────────┘    └─────────────┐
                 │                          │
                 ▼                          ▼
        ┌───────────────────┐      ┌──────────────────┐
        │ Duplicate Ticket  │      │ Prepare AI Input │
        │ Stop / Error      │      └─────────┬────────┘
        └───────────────────┘                │
                                             ▼
                                  ┌────────────────────────┐
                                  │ Classify the Ticket    │
                                  │                        │
                                  │ + Classifier Model     │
                                  │ + Structured Output    │
                                  │   Parser               │
                                  └───────────┬────────────┘
                                              │
                                              ▼
                                  ┌────────────────────────┐
                                  │ Validate AI Output     │
                                  │ Deterministic JS       │
                                  └───────────┬────────────┘
                                              │
                                              ▼
                                   ┌──────────────────────┐
                                   │ AI Output Valid?     │
                                   └─────────┬────────────┘
                                      YES    │    NO
                            ┌────────────────┘     └────────────────┐
                            │                                       │
                            ▼                                       ▼
                ┌───────────────────────┐              ┌──────────────────────┐
                │ Human Review Router   │              │ Prepare AI Fallback  │
                │ Deterministic JS      │              └──────────┬───────────┘
                └───────────┬───────────┘                         │
                            │                                     ▼
                            ▼                         ┌────────────────────────┐
                ┌──────────────────────┐              │ AI Processing Errors   │
                │ Needs Human Review?  │              │ Log / Manual Fallback  │
                └──────────┬───────────┘              └────────────────────────┘
                           │
                 YES       │       NO
              ┌────────────┘        └─────────────────┐
              │                                       │
              ▼                                       ▼
┌──────────────────────────────┐          ┌─────────────────────────┐
│ Prepare Human Review Queue   │          │ Route Support Queue     │
│                              │          │                         │
│ assigned_queue: HUMAN_REVIEW │          │ Switch by category      │
│ status: PENDING_REVIEW       │          └────────────┬────────────┘
│ final_route: HUMAN_REVIEW    │                       │
└──────────────┬───────────────┘        ┌──────────────┼──────────────┬──────────────┐
               │                        │              │              │              │
               │                        ▼              ▼              ▼              ▼
               │                    BILLING        TECHNICAL       ACCOUNT         SALES
               │                        │              │              │              │
               │                        └──────────────┴──────┬───────┴──────────────┘
               │                                             │
               └──────────────────────────┬──────────────────┘
                                          │
                                          ▼
                             ┌─────────────────────────────┐
                             │ Normalize Response Context  │
                             └──────────────┬──────────────┘
                                            │
                                            ▼
                             ┌──────────────────────────────┐
                             │ Generate Suggested Response  │
                             │                              │
                             │ + Response Model             │
                             │ + Safety-Constrained Prompt  │
                             └──────────────┬───────────────┘
                                            │
                                            ▼
                             ┌─────────────────────────────┐
                             │ Prepare Final Ticket Record │
                             └──────────────┬──────────────┘
                                            │
                                            ▼
                             ┌─────────────────────────────┐
                             │ Final Ticket Record         │
                             │ Delivery                    │
                             │                             │
                             │ Google Sheets → Tickets     │
                             └─────────────────────────────┘
```

---

## Simplified System Flow

```text
Receive
→ Normalize
→ Validate
→ Deduplicate
→ AI Interpret
→ Validate AI
→ Apply Business Policy
→ Route
→ Draft Safely
→ Persist
```

The workflow therefore contains multiple control boundaries before and after AI processing.

---

# 1. Input Layer

## Webhook

The workflow begins with an HTTP `POST` webhook.

The webhook receives support-ticket payloads from external sources such as:

- support forms
- email integrations
- APIs

A typical payload contains:

```json
{
  "ticket_id": "TKT-1001",
  "customer_name": "Example Customer",
  "customer_email": "customer@example.com",
  "company": "Example Company",
  "message": "I was charged twice for my subscription.",
  "source": "support_form"
}
```

The webhook only receives the request.

No business decision is made at this stage.

---

# 2. Normalization Layer

## Normalize Support Ticket

Raw webhook data is converted into a consistent internal ticket structure.

The workflow normalizes fields such as:

```text
ticket_id
customer_name
customer_email
company
customer_message
source
received_at
```

Normalization includes operations such as:

- trimming whitespace
- normalizing email casing
- standardizing customer/company text
- renaming `message` to `customer_message`
- generating a processing timestamp

This creates a predictable contract for downstream nodes.

---

# 3. Input Validation Layer

## Validate Support Ticket

Before any AI call occurs, the workflow validates the incoming payload deterministically.

Validation checks include:

### Ticket ID

```text
ticket_id must not be empty
```

### Customer Email

The email must match a valid email format.

### Customer Message

```text
customer_message must not be empty
```

### Source

The source must belong to the allowed set:

```text
support_form
email
api
```

AI is never used to determine whether the payload itself is valid.

---

## Invalid Input Path

If validation fails:

```text
Validate Support Ticket
        ↓
Prepare Rejected Ticket
        ↓
Append Rejected Tickets
```

The failure is recorded as:

```text
failure_type  = INVALID_INPUT
failure_stage = INPUT_VALIDATION
```

Invalid tickets never reach the classifier.

---

# 4. Idempotency / Duplicate Protection

## Find Existing Ticket

Valid tickets are checked against the existing `Tickets` dataset using:

```text
Ticket ID
```

as the idempotency key.

The Google Sheets lookup uses `Always Output Data` so the workflow continues even when no existing ticket is found.

---

## Duplicate Ticket?

If an existing Ticket ID is found:

```text
Duplicate Ticket? = TRUE
```

the workflow moves to the duplicate stop node.

```text
Find Existing Ticket
        ↓
Duplicate Ticket?
        ↓
Duplicate Ticket Stop
```

No classifier call occurs.

No response is generated.

No second ticket record is persisted.

This protects the workflow from:

- duplicate records
- unnecessary model calls
- repeated processing
- repeated downstream actions

---

# 5. AI Input Boundary

## Prepare AI Input

Only the data required by the classifier is passed into the AI layer.

The classifier receives:

```text
ticket_id
customer_message
prompt_version
```

The evaluated prompt version is:

```text
support-triage-v2
```

This follows a data-minimization approach by avoiding unnecessary customer information inside the classification prompt.

---

# 6. AI Classification Layer

## Classify the Ticket

The first LLM performs language interpretation.

It converts the customer's unstructured message into structured operational information.

The model produces exactly five fields:

```json
{
  "category": "billing",
  "summary": "Customer reports a duplicate subscription charge.",
  "requested_action": "Refund the duplicate charge.",
  "urgency": "medium",
  "needs_review": false
}
```

---

## Category

Allowed categories:

```text
billing
technical
account
sales
other
```

---

## Summary

A concise representation of the customer's issue.

The model is instructed not to invent missing information.

---

## Requested Action

Captures the customer's explicit request.

Examples:

```text
Refund the duplicate charge.
Reset the customer's password.
Provide Pro plan pricing information.
```

If no legitimate action is clearly requested:

```json
"requested_action": null
```

---

## Urgency

Allowed values:

```text
low
medium
high
```

The model interprets urgency using defined operational boundaries instead of customer emotion alone.

---

## needs_review

`needs_review` represents **interpretation uncertainty only**.

It does not authorize human review policy.

Examples that may produce:

```json
"needs_review": true
```

include:

- missing important context
- unclear customer intent
- genuinely ambiguous categories
- no legitimate support issue

Sensitive business rules are handled later outside the model.

---

# 7. Structured Output Enforcement

## Structured Output Parser

The classifier is connected to an n8n Structured Output Parser.

The parser enforces a JSON schema containing:

```text
category
summary
requested_action
urgency
needs_review
```

Rules include:

```text
category         → allowed enum
summary          → non-empty string
requested_action → string or null
urgency          → low | medium | high
needs_review     → boolean
```

Additional properties are not allowed.

This creates the first structural boundary around the model output.

---

# 8. Deterministic AI Output Validation

## Validate AI Output

The system does not assume that model output is safe simply because a parser exists.

A JavaScript validation layer independently verifies the AI response.

It checks:

### category

Must be one of:

```text
billing
technical
account
sales
other
```

### summary

Must:

```text
exist
be a string
not be empty
```

### requested_action

Must be:

```text
string
OR
null
```

### urgency

Must be one of:

```text
low
medium
high
```

### needs_review

Must be a strict boolean:

```text
true
false
```

The validator returns:

```text
ai_output_valid
ai_validation_errors
```

---

# 9. Invalid AI Output Fallback

## AI Output Valid?

Valid AI output proceeds to business policy.

Invalid AI output follows a separate failure path:

```text
Validate AI Output
        ↓
AI Output Valid?
        ↓ FALSE
Prepare AI Fallback
        ↓
AI Processing Errors
```

The fallback state uses:

```text
failure_type      = INVALID_AI_OUTPUT
failure_stage     = AI_OUTPUT_VALIDATION
processing_status = MANUAL_FALLBACK
```

Invalid model output never reaches support routing.

This separates:

```text
AI interpretation failure
```

from:

```text
business routing
```

---

# 10. Deterministic Human Review Policy

## Human Review Router

Once AI output is structurally valid, deterministic JavaScript rules determine whether automatic processing is allowed.

This layer is intentionally separate from the AI classifier.

The model interprets the message.

The workflow decides whether the ticket requires human control.

---

## Review Rule: AI Uncertainty

If:

```text
needs_review === true
```

the ticket requires human review.

---

## Review Rule: Refund Requests

If `requested_action` contains:

```text
refund
```

human approval is required.

Example reason:

```text
Refund request requires human approval
```

---

## Review Rule: Account Security

Account tickets are scanned for security-related concepts such as:

```text
ownership
compromise
compromised
unauthorized
hacked
takeover
access issue
security
```

If detected, human review is required.

---

## Review Rule: High Urgency

If:

```text
urgency = high
```

the ticket requires human review.

---

## Review Rule: Other Category

If:

```text
category = other
```

the workflow sends the ticket for human review rather than automatically assigning it to a specialized support queue.

---

## Human Review Output

The router adds:

```text
needs_human_review
review_reasons
```

This creates an important distinction:

```text
AI needs_review
        ≠
Business needs_human_review
```

The first represents interpretation uncertainty.

The second represents deterministic operational policy.

---

# 11. Human Review Branch

## Needs Human Review?

If:

```text
needs_human_review = true
```

the workflow enters:

```text
Prepare Human Review Queue
```

and sets:

```text
assigned_queue    = HUMAN_REVIEW
processing_status = PENDING_REVIEW
final_route       = HUMAN_REVIEW
```

The AI does not approve or execute the sensitive action.

---

# 12. Automated Support Queue Routing

Tickets that do not require human review continue to the support queue router.

## Route Support Queue

The deterministic Switch node routes tickets according to:

```text
category
```

Available queues:

```text
BILLING
TECHNICAL
ACCOUNT
SALES
```

Each branch assigns:

```text
assigned_queue
processing_status
final_route
```

Example:

```text
category          = technical
assigned_queue    = TECHNICAL
processing_status = READY_FOR_RESPONSE
final_route       = TECHNICAL
```

---

# 13. Response Context Normalization

## Normalize Response Context

Human-review tickets and automatically routed tickets converge before response drafting.

This node creates a shared response context containing:

```text
assigned_queue
processing_status
final_route
customer_message
```

This prevents the response generator from needing to know which upstream branch executed.

---

# 14. Suggested Response Generation

## Generate Suggested Response

A second LLM generates a short customer-facing response draft.

This model has a different responsibility from the classifier.

The classifier interprets operational meaning.

The responder produces language.

---

## Trusted Response Context

The response model receives trusted workflow fields such as:

```text
category
summary
requested_action
needs_human_review
customer_message
```

---

# 15. Response Safety Boundary

The responder is explicitly restricted from claiming actions that the workflow has not confirmed.

It must not falsely claim that:

- a refund was processed
- an investigation started
- a ticket was escalated
- an issue was fixed
- an action was approved
- a specialist was assigned
- a reviewer started work
- future contact is guaranteed
- a resolution timeline exists

A fundamental response rule is:

> **Routing a ticket to a queue does not mean work has started.**

---

## Normal Routed Tickets

For normal tickets, the response should primarily:

1. acknowledge the issue
2. restate the customer's problem or request
3. avoid promises or unconfirmed actions

---

## Human Review Tickets

If:

```text
needs_human_review = true
```

the draft may only:

1. acknowledge the issue
2. acknowledge the requested action
3. state that the request requires review

It must not predict:

- what the reviewer will do
- whether the action will be approved
- when the issue will be resolved

---

# 16. Final Ticket Record

## Prepare Final Ticket Record

After response generation, the workflow constructs the final operational record.

The record contains:

### Original Ticket Data

```text
ticket_id
received_at
customer_name
customer_email
company
source
customer_message
```

### AI Interpretation

```text
category
summary
requested_action
urgency
ai_needs_review
```

### Deterministic Policy Result

```text
needs_human_review
review_reasons
assigned_queue
```

### Response Data

```text
suggested_response
```

### Processing State

```text
processing_status
final_route
prompt_version
processed_at
```

---

# 17. Persistence Layer

## Final Ticket Record Delivery

Successful ticket records are appended to the main Google Sheets `Tickets` dataset.

The spreadsheet acts as the project's lightweight operational datastore.

Additional sheets are used for separate failure and evaluation concerns.

Examples include:

```text
Tickets
Rejected Tickets
AI Processing Errors
Evaluation Results
Evaluation Test Cases
Urgency Edge Cases
Urgency Edge Results
```

This separation makes successful processing, failures, and AI evaluation independently observable.

---

# Failure Architecture

The workflow intentionally separates different failure types.

## Invalid Input

```text
Incoming Payload
      ↓
Normalize
      ↓
Validate
      ↓ INVALID
Prepare Rejected Ticket
      ↓
Rejected Tickets
```

State:

```text
INVALID_INPUT
INPUT_VALIDATION
```

---

## Duplicate Ticket

```text
Valid Ticket
     ↓
Find Existing Ticket
     ↓
Duplicate Ticket?
     ↓ YES
Stop Processing
```

The ticket never reaches AI processing.

---

## Invalid AI Output

```text
AI Classification
      ↓
Validate AI Output
      ↓ INVALID
Prepare AI Fallback
      ↓
AI Processing Errors
```

State:

```text
INVALID_AI_OUTPUT
AI_OUTPUT_VALIDATION
MANUAL_FALLBACK
```

---

# Control Boundaries

The workflow contains several independent safeguards.

```text
                    ┌─────────────────────────┐
                    │ Incoming Customer Data  │
                    └────────────┬────────────┘
                                 │
                    ─────────────▼─────────────
                    INPUT VALIDATION BOUNDARY
                    ─────────────┬─────────────
                                 │
                    ─────────────▼─────────────
                    IDEMPOTENCY BOUNDARY
                    ─────────────┬─────────────
                                 │
                                 ▼
                         AI INTERPRETATION
                                 │
                    ─────────────▼─────────────
                    AI OUTPUT VALIDATION
                    ─────────────┬─────────────
                                 │
                    ─────────────▼─────────────
                    BUSINESS POLICY BOUNDARY
                    ─────────────┬─────────────
                                 │
                                 ▼
                         QUEUE ROUTING
                                 │
                                 ▼
                         AI RESPONSE DRAFT
                                 │
                    ─────────────▼─────────────
                    RESPONSE SAFETY BOUNDARY
                    ─────────────┬─────────────
                                 │
                                 ▼
                            PERSISTENCE
```

No single AI output directly controls the entire workflow.

---

# AI vs Deterministic Responsibilities

| Responsibility | AI | Deterministic Workflow |
|---|:---:|:---:|
| Understand customer language | ✅ | |
| Summarize ticket | ✅ | |
| Extract requested action | ✅ | |
| Estimate urgency | ✅ | |
| Identify interpretation uncertainty | ✅ | |
| Draft customer response | ✅ | |
| Validate incoming payload | | ✅ |
| Enforce duplicate protection | | ✅ |
| Validate AI schema and types | | ✅ |
| Approve refund handling | | ✅ |
| Enforce security review | | ✅ |
| Enforce high-urgency review | | ✅ |
| Decide support queue | | ✅ |
| Set processing state | | ✅ |
| Persist records | | ✅ |
| Handle invalid AI output | | ✅ |

The model provides interpretation.

The workflow retains operational control.

---

# System States

The architecture produces several meaningful processing states.

## Ready for Response

```text
processing_status = READY_FOR_RESPONSE
```

Used when the ticket:

- passed validation
- passed duplicate protection
- produced valid AI output
- did not trigger human-review policy
- was routed to a normal support queue

---

## Pending Review

```text
processing_status = PENDING_REVIEW
```

Used when deterministic policy requires human intervention.

---

## Manual Fallback

```text
processing_status = MANUAL_FALLBACK
```

Used when the AI output fails deterministic validation.

---

## Rejected

Invalid incoming tickets are logged separately instead of entering the processing pipeline.

---

# Prompt Versioning

The classifier prompt is versioned.

The final evaluated version used by the workflow is:

```text
support-triage-v2
```

The prompt version is stored with processed ticket records.

This makes it possible to:

- compare future prompt versions
- perform regression testing
- trace classifier behavior
- reproduce evaluation results

---

# Data Flow Summary

```text
Raw Ticket
   │
   ▼
Normalized Ticket
   │
   ▼
Validated Ticket
   │
   ▼
Unique Ticket
   │
   ▼
Minimal AI Input
   │
   ▼
Structured AI Interpretation
   │
   ▼
Validated AI Interpretation
   │
   ▼
Deterministic Review Decision
   │
   ▼
Operational Queue State
   │
   ▼
Safe Response Draft
   │
   ▼
Final Ticket Record
   │
   ▼
Persistent Operational Data
```

---

# Design Decisions

## Why Not Let the AI Route Everything?

LLMs are useful for interpreting language but can be:

- inconsistent
- probabilistic
- vulnerable to adversarial input
- incorrect about business policy

Therefore, classification and extraction can use AI while consequential routing rules remain deterministic.

---

## Why Validate AI Output Twice?

The Structured Output Parser provides schema enforcement at the model boundary.

The JavaScript validator provides an independent deterministic verification layer.

This creates defense in depth.

---

## Why Separate `needs_review` and `needs_human_review`?

They represent different concepts.

```text
needs_review
```

means:

> The AI is uncertain about its interpretation.

```text
needs_human_review
```

means:

> Business policy requires human intervention.

A ticket can therefore be:

```text
AI interpretation = confident
Business policy    = human review required
```

For example, a refund request may be completely clear while still requiring human approval.

---

## Why Use a Separate Response Model?

Classification and response drafting have different responsibilities.

The classifier produces strict structured operational data.

The responder produces natural language.

Separating them reduces prompt complexity and allows different safety constraints for each task.

---

## Why Generate Drafts Instead of Taking Actions?

The workflow intentionally produces:

```text
suggested_response
```

rather than allowing the AI to execute customer-impacting actions.

This keeps the system useful for support operations without giving the model authority to:

- refund money
- delete accounts
- change ownership
- approve requests
- modify subscriptions
- make promises on behalf of the company

---

# Final Architecture Principle

The architecture is built around controlled AI assistance rather than autonomous AI decision-making.

```text
AI
│
├── Understand
├── Summarize
├── Extract
├── Estimate
└── Draft
        │
        ▼
Deterministic Workflow
│
├── Validate
├── Verify
├── Apply Policy
├── Route
├── Control State
├── Handle Failure
└── Persist
```

The result is an AI-assisted support workflow where language interpretation remains flexible while operational control remains predictable, testable, and deterministic.
