# Workflow Architecture

## Overview

The **AI Support Operations Triage System** separates AI interpretation from deterministic workflow control.

> **AI interprets. Deterministic systems control.**

The LLM understands unstructured customer messages, while n8n and JavaScript handle validation, duplicate prevention, AI-output verification, human-review policies, queue routing, failure handling, and persistence.

---

## Full Workflow Blueprint

```text
                              ┌─────────────────────┐
                              │   Incoming Ticket   │
                              └──────────┬──────────┘
                                         │
                                         ▼
                                   ┌───────────┐
                                   │  Webhook  │
                                   └─────┬─────┘
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
                                     │
                       ┌─────────────┴──────────────┐
                       │                            │
                    INVALID                       VALID
                       │                            │
                       ▼                            ▼
          ┌─────────────────────────┐    ┌──────────────────────┐
          │ Prepare Rejected Ticket │    │ Find Existing Ticket │
          └────────────┬────────────┘    └──────────┬───────────┘
                       │                            │
                       ▼                            ▼
          ┌─────────────────────────┐    ┌──────────────────────┐
          │ Append Rejected Tickets │    │  Duplicate Ticket?   │
          └─────────────────────────┘    └──────────┬───────────┘
                                                   │
                                    ┌──────────────┴─────────────┐
                                    │                            │
                                 DUPLICATE                      NEW
                                    │                            │
                                    ▼                            ▼
                          ┌──────────────────┐          ┌──────────────────┐
                          │ Stop Processing  │          │ Prepare AI Input │
                          └──────────────────┘          └─────────┬────────┘
                                                                  │
                                                                  ▼
                                                       ┌─────────────────────┐
                                                       │ Classify the Ticket │
                                                       │                     │
                                                       │ + Classifier Model  │
                                                       │ + Structured Parser │
                                                       └──────────┬──────────┘
                                                                  │
                                                                  ▼
                                                       ┌────────────────────┐
                                                       │ Validate AI Output │
                                                       │   JavaScript       │
                                                       └──────────┬─────────┘
                                                                  │
                                                                  ▼
                                                       ┌────────────────────┐
                                                       │ AI Output Valid?   │
                                                       └──────────┬─────────┘
                                                                  │
                                          ┌───────────────────────┴──────────────────────┐
                                          │                                              │
                                        INVALID                                        VALID
                                          │                                              │
                                          ▼                                              ▼
                              ┌─────────────────────┐                       ┌─────────────────────┐
                              │ Prepare AI Fallback │                       │ Human Review Router │
                              └──────────┬──────────┘                       │    JavaScript       │
                                         │                                  └──────────┬──────────┘
                                         ▼                                             │
                              ┌─────────────────────┐                                   ▼
                              │ AI Processing       │                       ┌─────────────────────┐
                              │ Errors              │                       │ Needs Human Review? │
                              └─────────────────────┘                       └──────────┬──────────┘
                                                                                     │
                                                              ┌──────────────────────┴───────────────────┐
                                                              │                                          │
                                                             YES                                         NO
                                                              │                                          │
                                                              ▼                                          ▼
                                                 ┌────────────────────────────┐              ┌─────────────────────┐
                                                 │ Prepare Human Review Queue │              │ Route Support Queue │
                                                 │                            │              └──────────┬──────────┘
                                                 │ HUMAN_REVIEW               │                         │
                                                 │ PENDING_REVIEW             │            ┌────────────┼────────────┬────────────┐
                                                 └─────────────┬──────────────┘            │            │            │            │
                                                               │                       BILLING     TECHNICAL     ACCOUNT       SALES
                                                               │                            │            │            │            │
                                                               │                            └────────────┴──────┬─────┴────────────┘
                                                               │                                               │
                                                               └──────────────────────┬────────────────────────┘
                                                                                      │
                                                                                      ▼
                                                                        ┌────────────────────────────┐
                                                                        │ Normalize Response Context │
                                                                        └─────────────┬──────────────┘
                                                                                      │
                                                                                      ▼
                                                                        ┌────────────────────────────┐
                                                                        │ Generate Suggested Response│
                                                                        │                            │
                                                                        │ + Responder Model          │
                                                                        │ + Safety Rules             │
                                                                        └─────────────┬──────────────┘
                                                                                      │
                                                                                      ▼
                                                                        ┌────────────────────────────┐
                                                                        │ Prepare Final Ticket Record│
                                                                        └─────────────┬──────────────┘
                                                                                      │
                                                                                      ▼
                                                                        ┌────────────────────────────┐
                                                                        │ Persist to Tickets Sheet   │
                                                                        └────────────────────────────┘
```

---

## Simplified Flow

```text
Receive
  ↓
Normalize
  ↓
Validate Input
  ↓
Check Duplicate
  ↓
AI Interpretation
  ↓
Validate AI Output
  ↓
Apply Business Policy
  ↓
Route Ticket
  ↓
Generate Safe Draft
  ↓
Persist Record
```

---

## Control Boundaries

```text
Customer Input
     │
     ▼
┌───────────────────────────┐
│ 1. INPUT VALIDATION       │
│                           │
│ Required fields           │
│ Email format              │
│ Allowed source            │
└────────────┬──────────────┘
             │
             ▼
┌───────────────────────────┐
│ 2. IDEMPOTENCY            │
│                           │
│ Existing Ticket ID?       │
│ Duplicate → Stop          │
└────────────┬──────────────┘
             │
             ▼
        ┌─────────┐
        │   AI    │
        │Interpret│
        └────┬────┘
             │
             ▼
┌───────────────────────────┐
│ 3. AI OUTPUT VALIDATION   │
│                           │
│ Schema                    │
│ Types                     │
│ Allowed enums             │
│ Required fields           │
└────────────┬──────────────┘
             │
             ▼
┌───────────────────────────┐
│ 4. BUSINESS POLICY        │
│                           │
│ Refund review             │
│ Security review           │
│ High urgency              │
│ AI uncertainty            │
│ Other category            │
└────────────┬──────────────┘
             │
             ▼
       Queue Routing
             │
             ▼
       ┌───────────┐
       │ Response  │
       │    AI     │
       └─────┬─────┘
             │
             ▼
┌───────────────────────────┐
│ 5. RESPONSE SAFETY        │
│                           │
│ No fake actions           │
│ No fake escalation        │
│ No fake timelines         │
│ No unsupported promises   │
└────────────┬──────────────┘
             │
             ▼
        Persistence
```

---

## AI vs Deterministic Responsibilities

```text
┌─────────────────────────┐       ┌────────────────────────────┐
│           AI            │       │  DETERMINISTIC WORKFLOW    │
├─────────────────────────┤       ├────────────────────────────┤
│ Interpret message       │       │ Validate input             │
│ Classify category       │       │ Detect duplicates          │
│ Summarize issue         │       │ Validate AI output         │
│ Extract request         │       │ Apply human-review rules   │
│ Estimate urgency        │       │ Route support queues       │
│ Detect uncertainty      │       │ Set processing states      │
│ Draft response          │       │ Handle failures            │
│                         │       │ Persist records            │
└─────────────────────────┘       └────────────────────────────┘
```

The model is responsible for **language understanding**.

The workflow remains responsible for **operational decisions**.

---

## Human Review Decision

```text
                    Valid AI Output
                           │
                           ▼
                 Human Review Router
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
     AI uncertain      Refund request   Security concern
          │                │                │
          └────────────────┼────────────────┘
                           │
                  High urgency / Other
                           │
                           ▼
                needs_human_review
                     TRUE / FALSE
                           │
              ┌────────────┴────────────┐
              │                         │
             TRUE                      FALSE
              │                         │
              ▼                         ▼
        HUMAN_REVIEW              Category Queue
        PENDING_REVIEW       BILLING / TECHNICAL /
                              ACCOUNT / SALES
```

This keeps:

```text
AI needs_review
```

separate from:

```text
Business needs_human_review
```

---

## Failure Paths

### Invalid Input

```text
Incoming Ticket
      ↓
Input Validation
      ↓
    INVALID
      ↓
Rejected Tickets

failure_type  = INVALID_INPUT
failure_stage = INPUT_VALIDATION
```

### Duplicate Ticket

```text
Valid Ticket
     ↓
Existing Ticket Lookup
     ↓
Duplicate Found
     ↓
Stop Processing
```

### Invalid AI Output

```text
AI Classification
      ↓
AI Output Validation
      ↓
    INVALID
      ↓
Manual Fallback
      ↓
AI Processing Errors

failure_type      = INVALID_AI_OUTPUT
failure_stage     = AI_OUTPUT_VALIDATION
processing_status = MANUAL_FALLBACK
```

---

## Processing States

```text
┌────────────────────┬──────────────────────────────────────────┐
│ STATE              │ MEANING                                  │
├────────────────────┼──────────────────────────────────────────┤
│ READY_FOR_RESPONSE │ Normal ticket routed successfully        │
│ PENDING_REVIEW     │ Human approval/review required           │
│ MANUAL_FALLBACK    │ AI output failed deterministic validation│
│ REJECTED           │ Incoming payload failed validation       │
└────────────────────┴──────────────────────────────────────────┘
```

---

## Data Transformation

```text
Raw Webhook Payload
        ↓
Normalized Ticket
        ↓
Validated Ticket
        ↓
Minimal AI Input
        ↓
Structured AI Output
        ↓
Validated AI Output
        ↓
Business Review Decision
        ↓
Queue + Processing State
        ↓
Safe Suggested Response
        ↓
Final Ticket Record
```

---

## Final Architecture Principle

```text
                FLEXIBLE
                   │
                   ▼
            AI Interpretation
                   │
                   ▼
        ───────────────────────
          Deterministic Guard
        ───────────────────────
                   │
                   ▼
            Business Policy
                   │
                   ▼
        ───────────────────────
          Deterministic Guard
        ───────────────────────
                   │
                   ▼
          AI Response Draft
                   │
                   ▼
        ───────────────────────
            Safety Boundary
        ───────────────────────
                   │
                   ▼
               Record
```

The workflow uses AI where language interpretation provides value while keeping validation, sensitive decisions, routing, and state management predictable and testable.
