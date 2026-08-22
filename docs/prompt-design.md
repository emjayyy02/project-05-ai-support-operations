# Prompt Design

## Overview

The classifier prompt evolved from a simple structured support-ticket prompt into a more controlled version based on measured evaluation failures.

The final evaluated classifier version is:

```text
support-triage-v2
```

The prompt is responsible for producing:

```text
category
summary
requested_action
urgency
needs_review
```

in structured JSON.

---

## Version 1

The original prompt defined:

- supported categories
- summary behavior
- requested-action extraction
- three urgency levels
- `needs_review` for ambiguity
- basic protection against instructions inside customer messages

V1 provided a solid baseline, but evaluation exposed several weaknesses:

- urgency was too loosely defined
- multi-intent tickets were inconsistent
- `needs_review` sometimes mixed ambiguity with sensitivity
- adversarial instructions could leak into `requested_action`
- multiple explicit requests were sometimes only partially extracted

---

## Version 2

`support-triage-v2` was created from those measured failures.

### 1. Clearer `needs_review`

V2 defines `needs_review` as **interpretation uncertainty only**.

It may be true when:

- important information is missing
- the issue cannot be confidently understood
- categories are genuinely ambiguous
- no clear legitimate support issue exists

It should not become true simply because a ticket is:

- financial
- security-related
- high urgency
- refund-related
- GDPR-related

Those concerns are handled later by deterministic workflow rules.

---

### 2. Stronger Urgency Calibration

V2 introduced clearer boundaries:

```text
LOW
Informational, minor, or no active disruption.

MEDIUM
Active customer problem with inconvenience or partial feature impact.

HIGH
Severe active impact such as major outage, unauthorized financial activity,
security compromise, data exposure, or serious time-sensitive business harm.
```

The prompt also explicitly prevents emotion alone from creating HIGH urgency.

---

### 3. Better Multi-Intent Handling

For tickets with multiple requests, V2 instructs the model to:

1. identify the dominant business issue
2. classify using that dominant category when possible
3. use `needs_review=true` only when the intents create real ambiguity
4. preserve all important explicit requested actions

Multiple intents alone do not automatically require human review.

---

### 4. Requested Action Constraints

`requested_action` must always be:

```text
one concise string
OR
null
```

It must never be returned as:

```text
array
object
list
```

This keeps the output compatible with the workflow schema.

---

### 5. Prompt Injection Protection

Customer messages are treated as untrusted data.

V2 explicitly rejects:

- prompt-injection instructions
- role-change instructions
- system-like commands
- output-format manipulation
- attempts to control classification fields

These instructions must not become `requested_action`.

However, legitimate support requests inside the same message are still extracted normally.

---

## Prompt Versioning

Each processed ticket stores:

```text
prompt_version = support-triage-v2
```

This allows future evaluation and regression testing to compare classifier behavior across prompt revisions.

---

## Design Principle

The prompt is intentionally limited to interpretation.

It does not decide business authorization.

```text
AI prompt
→ interpret the ticket

Deterministic workflow
→ validate
→ apply policy
→ decide human review
→ route
```

This separation keeps the language model useful without making it the final authority over business actions.
