# Evaluation

## Overview

The classifier was evaluated with a fixed support-ticket benchmark instead of relying only on successful demo runs.

The goal was to identify real failure patterns, improve the prompt intentionally, and verify whether the changes actually helped.

---

## Test Set

The main benchmark contained **20 support tickets** covering:

- billing
- technical issues
- account problems
- sales inquiries
- ambiguous messages
- multi-intent requests
- security-related tickets
- long-form tickets
- prompt injection attempts

The expected fields were:

```text
category
urgency
requested_action
needs_review
```

---

## Evaluation Method

These fields were graded automatically:

```text
category
urgency
needs_review
```

`requested_action` was reviewed manually for semantic equivalence because correct outputs can use different wording.

Example:

```text
Expected:
refund duplicate charge

Actual:
Refund the duplicate charge of $49.99

Result:
PASS
```

---

## Version 1 Findings

The first classifier version exposed several weaknesses:

- urgency was inconsistently calibrated
- multi-intent tickets were harder to classify
- sensitive tickets sometimes triggered `needs_review` even when interpretation was clear
- prompt-injection text could leak into `requested_action`
- some multi-action requests were only partially extracted

These findings were used to create:

```text
support-triage-v2
```

---

## Version 2 Improvements

V2 introduced targeted changes for:

- clearer urgency definitions
- dominant-intent handling
- stricter `needs_review` meaning
- prompt-injection resistance
- multi-request extraction
- stronger requested-action constraints

The changes were based directly on measured V1 failures rather than random prompt expansion.

---

## Results

V2 achieved approximately:

```text
Category accuracy:      95%
needs_review accuracy:  85%
```

Requested-action extraction also improved significantly, especially for:

- multi-action tickets
- ambiguous requests
- prompt-injection cases

---

## Urgency Calibration

During review, several original expected urgency labels were found to conflict with the written V2 urgency policy.

Those benchmark labels were corrected before further prompt tuning to avoid overfitting the model to inconsistent expected answers.

After calibration, the classifier reached approximately:

```text
Urgency accuracy: 85%
```

---

## Fresh Urgency Edge Test

A separate set of **5 unseen urgency cases** was created after the V2 prompt was finalized.

Results:

```text
U1 → PASS
U2 → FAIL
U3 → PASS
U4 → PASS
U5 → PASS
```

Final score:

```text
4 / 5
80%
```

The remaining known weakness was occasional under-classification of active but non-severe billing discrepancies.

---

## Final Decision

The classifier was intentionally frozen at:

```text
support-triage-v2
```

rather than repeatedly tuning the prompt to chase perfect benchmark scores.

The evaluation process followed:

```text
Build
→ Measure
→ Identify Failure Patterns
→ Make Targeted Changes
→ Retest
→ Freeze
```

The objective was measurable and explainable improvement, not artificial 100% accuracy.
