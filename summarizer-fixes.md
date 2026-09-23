# Clinical Summary and Phase Analysis: Client Change Summary

## Purpose of this system

The platform turns a patient's clinical notes and VALD performance data into two clinician-facing outputs:

1. A **clinical summary** with active areas of concern, risks, treatment gaps, and a concise rehabilitation assessment.
2. A **phase analysis** that shows the patient's rehabilitation stage, changes over time, and protocol alerts.

The dashboard reads these generated records from MongoDB and presents them to clinicians. The backend keeps the output structure unchanged, so the existing frontend continues to work.

## Issue we identified

The previous summary process allowed the AI model to generate concern scores and risk labels, then stored those values with limited backend validation.

This created three risks:

| Area | Previous behaviour | Why it was a problem |
|---|---|---|
| Concern score | The AI supplied the final score. | A score could be inconsistent with its safety, severity, and urgency components. |
| Clinical evidence | A concern could be stored without a traceable date from the patient timeline. | Reviewers could not easily confirm which report or VALD session supported it. |
| VALD data | Zero-force values and session dates were not handled consistently. | Invalid readings could influence an assessment, or useful session context could be missing. |
| AI instructions | Several overlapping score instructions existed in the prompt. | This could produce inconsistent wording, labels, and priorities between patients. |

This did **not** mean every historic concern was wrong. It meant the backend did not provide enough deterministic checks to guarantee that each generated concern was traceable and internally consistent.

## What we changed

### 1. Backend-calculated concern scores

Each concern contains three components: safety, severity, and urgency. The backend now calculates the final score from those components using one fixed formula:

```text
criticality = (safety × 2.2 + severity × 2.0 + urgency × 1.3) ÷ 5.5
```

The backend also assigns the risk band from the calculated score:

| Score | Risk band |
|---:|---|
| 0–3 | Low |
| Above 3–6 | Moderate |
| Above 6–8 | High |
| Above 8–10 | Critical |

**Result:** the displayed score and risk band are now calculated consistently by code instead of being accepted solely from AI text.

### 2. Evidence and source-date validation

New concern instructions require dated evidence. The date must match a date in the patient's loaded clinical or VALD timeline.

The backend rejects a generated concern when it has missing required score components, an invalid score relationship, missing dated evidence, or an evidence date that is absent from the patient timeline.

**Result:** a generated concern must be linked to data that was actually loaded for that patient.

### 3. Safer score rules

The backend validates all component scores are finite values between 0 and 10. It also rejects implausible combinations, such as maximum urgency with low safety.

**Result:** malformed or contradictory AI output is not saved as a clinical concern.

### 4. Safer VALD data handling

Zero-force readings are now treated as invalid measurements and removed from the usable values while preserving other valid readings in the same session. The processing also prefers the VALD session dates supplied by the data transformer.

**Result:** invalid zero readings are less likely to distort force/asymmetry analysis, and the analysis has more reliable session timing.

### 5. Clearer AI output contract

The AI workflow now asks for a small number of active concerns only. Each must include safety, severity, urgency, and dated supporting evidence. It is instructed not to create a concern where the source is missing, laterality is unclear, or the issue is documented as resolved.

**Result:** the AI is guided to produce focused, auditable output before backend validation is applied.

### 6. Reliability improvements

The backend uses explicit Vertex AI project and location configuration, configured model fallback, output time limits, durable job status, idempotency, retries, cancellation protection, and separate summary/phase output handling.

**Result:** job execution is more predictable and duplicate or late writes are less likely.

### 7. Local service health fix

The phase-analysis health endpoint had a missing Python import. This was fixed and the local phase service health endpoint now responds successfully.

**Result:** the local development stack can be checked reliably before a release.

## What did not change

- The MongoDB document shape used by the dashboard was preserved.
- The frontend integration and its displayed fields remain compatible.
- Existing historical summaries were not automatically overwritten.
- No patient result was written during the approved local dry-run investigation.

## Validation completed

| Validation | Result |
|---|---|
| Automated backend tests | 101 passed; 13 database integration tests skipped because they require a dedicated test database. |
| New concern scoring rules | Passed targeted tests. |
| Evidence persistence checks | Passed targeted tests. |
| VALD zero-value sanitization | Passed targeted tests. |
| VALD timeline date handling | Passed targeted tests. |
| Local container health | Orchestrator, clinical-auditor, and phase-analysis services were healthy. |
| MongoDB writes during investigation | No generated patient output was written. |

## Current status and remaining work

| Work item | Status | Meaning |
|---|---|---|
| Deterministic scoring and validation | Complete | New generated concerns are checked before they can be persisted. |
| Evidence-date requirement | Complete | New concerns must cite a date from the loaded patient timeline. |
| VALD data safeguards | Complete | Zero-force handling and session-date use are improved. |
| Local service health | Complete | The local stack is running and health checks pass. |
| Clinical review of new AI output | Pending | A clinician should review a sample of regenerated summaries for clinical appropriateness. |
| Full end-to-end AI generation validation | Pending | The deployed dependency/model configuration must complete a full generated-summary test successfully. |
| Regeneration of historic production summaries | Pending approval | This should happen only after the sample review is accepted. |

## How to explain the result to a client

> We found that the prior system could save AI-generated concern scores without enough backend verification of the score calculation and clinical evidence. We have added deterministic score calculation, source-date validation, stricter concern rules, and safer VALD timeline handling. This makes new summaries more consistent and traceable. The backend corrections and local health checks are complete; the final step is a clinician-reviewed end-to-end sample before regenerating existing production summaries.

## Important limitation

These changes improve the engineering controls around AI output. They do not replace clinical judgement. A clinician must still review generated assessments, particularly when a concern involves a diagnosis, causation, or treatment decision.

## Recommended release path

1. Complete a no-write end-to-end generation test using the intended deployed Vertex AI dependency and model configuration.
2. Have a clinician compare a small representative sample of new summaries with source notes and VALD data.
3. Approve the output format and clinical quality.
4. Deploy the backend changes.
5. Regenerate historic summaries in controlled batches, monitor results, and retain rollback capability.

