# Clinical Summary Accuracy: Detailed Implementation Proposal

## Scope

This is a proposed implementation plan. It does not change production data or approve any clinical scoring rule. Doctors and the client must approve the clinical rules and examples before development begins.

The goal is to make every displayed Area of Concern traceable to patient data, scored consistently, protected from invalid VALD interpretation, and preserved correctly in the dashboard summary.

## Proposed implementation work

| Work area | Where we will implement it | What we will implement | Why it is needed | Expected result |
|---|---|---|---|---|
| Clinical scoring policy | `LLM/report-gen/clinical_guardrails.json` | Keep the agreed definitions, formula, score limits, risk bands, evidence rules, and inclusion/exclusion rules in one approved configuration file. | Scoring rules should not be repeated differently in several prompts or Python files. | One reviewed source of truth for the rules used by every new summary. |
| Detailed AI prompt | `LLM/report-gen/eba_agent.py` | Rewrite the prompt around the approved rules. Require each concern to include evidence, source date, affected side where relevant, and the latest relevant record. Prohibit concerns based only on missing data, programme critique, or unsupported inference. | The current broad prompt can allow incomplete or inconsistent clinical output. | The detailed report requests only evidence-based, clinically reviewable concerns. |
| Patient timeline | `LLM/shared/agent_data_pipeline.py`, `LLM/shared/chronological_data_processor.py`, `LLM/shared/load_reports_from_mongo.py` | Load and sort clinical records and VALD sessions chronologically. Pass enough patient history to establish baseline, latest status, and valid trends. | A summary is incorrect if it uses an old record as current or creates a trend from incomplete history. | The AI receives the correct patient sequence and can identify whether a finding is current, historical, improving, or unresolved. |
| VALD trial processing | `LLM/report-gen/vald_preparation.py`, `LLM/shared/fetch_and_transform_vald.py` | Use only valid recorded trials, calculate a session mean for each side, retain session date and exercise name, and calculate asymmetry from the raw bilateral values. | Selecting one peak trial or trusting a derived field can understate or overstate a deficit. | Every displayed VALD asymmetry can be traced to left/right values, session date, exercise, and calculation basis. |
| Incomplete VALD data | `LLM/report-gen/vald_preparation.py`, `LLM/shared/fetch_and_transform_vald.py` | Treat zero, null, blank, or absent limb readings as incomplete testing. Do not calculate or score an asymmetry from one-sided data. | Missing data can otherwise appear as a false 100% patient deficit. | The report requests a re-test or identifies a data gap instead of creating a false clinical concern. |
| Single-session rules | `LLM/report-gen/clinical_guardrails.json`, `LLM/report-gen/eba_agent.py` | Allow a valid single session to describe that day’s measurement, but prevent claims such as improving, persistent, regression, or plateau unless at least two comparable sessions exist. | One measurement cannot establish a patient trajectory. | Reports distinguish a measured deficit from a trend claim and avoid overstating progression. |
| Deterministic score validation | `LLM/report-gen/scoring_utils.py`, `LLM/report-gen/eba_validator.py`, `LLM/report-gen/push_report_to_mongo.py` | Recalculate criticality from Safety, Severity, and Urgency; verify score range, formula arithmetic, risk band, evidence requirement, and approved score relationships before persistence. | AI text alone must not be the final authority for displayed scores. | Incorrect arithmetic, invalid score combinations, and unsupported high-risk scores are blocked before saving. |
| Detailed-to-concise fidelity | `LLM/report-gen/concise_agent.py`, `LLM/report-gen/concise_validator.py` | Make the dashboard summary preserve the approved concern heading, component scores, final score, evidence, date, and important measurements from the detailed report. Compare both outputs before persistence. | A valid detailed report can become inaccurate if the concise step drops or changes information. | Dashboard output matches the approved detailed assessment for every scored concern. |
| Publication gate and job result | `LLM/report-gen/run_audit_by_stance_id.py`, `api/summary_api.py`, `api/job_store.py` | Treat validation failure, incomplete mandatory sections, or failed persistence as a failed job. Do not write a summary when validation fails. Record a safe reason for the failure. | A job must not show as completed if its output was blocked or not saved. | Operators can see which jobs need review, and invalid output cannot silently reach the dashboard. |
| Development isolation | `docker-compose.dev.yml`, `.env.dev` | Keep development containers, ports, queue records, and output collections separate. Use `new-summary-dev` only for comparison work. | Testing must not overwrite clinician-facing production summaries. | Development output is safe to review alongside production output. |
| Secret and artifact protection | `.gitignore`, `.dockerignore`, Docker Compose files, deployment environment configuration | Remove credentials and generated patient reports from tracked files and Docker build contexts. Load credentials through protected environment configuration or mounted secrets. | Credentials and patient reports must never be committed or included in an image by mistake. | Safer source control and deployment process. |
| Automated and clinical tests | `LLM/report-gen/tests/`, `tests/`, development test scripts | Add unit tests for formulas, risk bands, evidence validation, null/zero values, one-sided data, session selection, timeline ordering, concise fidelity, and job failure status. Run approved patient examples in development. | The rules need repeatable verification after any future change. | Changes that reintroduce a known error fail before release; doctors can review real development output. |

## Proposed clinical scoring model

The candidate formula is:

```text
Criticality = (Safety × 2.2 + Severity × 2.0 + Urgency × 1.3) ÷ 5.5
```

This is a proposed engineering implementation, not an approved clinical rule. The doctors must approve or change the definitions, weights, score ranges, risk bands, and required evidence before it is used for production summaries.

## Development and approval process

| Stage | What we will do | Result before moving forward |
|---|---|---|
| 1. Clinical rule approval | Review the concern rules, score definitions, formula, VALD rules, and example scenarios with doctors. | Written approval of the clinical rules. |
| 2. Backend implementation | Apply the approved rules in the prompt, data pipeline, validators, persistence gate, and tests. | Code review and automated-test results. |
| 3. Development comparison | Generate approved representative patients into `new-summary-dev`. Compare current and proposed summaries against source records. | Side-by-side review pack for doctors. |
| 4. Clinical acceptance | Doctors review concern inclusion, score, evidence, date, and dashboard wording. | Sign-off on sample output. |
| 5. Controlled release | Deploy the approved version and run a small monitored batch before wider regeneration. | Production acceptance before full rollout. |

## What will be visible after implementation

For each Area of Concern, the intended visible result is:

- a clear clinical finding;
- Safety, Severity, Urgency, and a backend-validated final score;
- source evidence and date;
- VALD exercise, session date, and left/right values when VALD is used;
- no score created solely because a test is missing;
- no false asymmetry from a missing side;
- no trend statement where only one comparable session exists;
- the same approved concern and score in the detailed report and the dashboard summary.

## Approval requested

Please approve the proposed direction for clinical review. Before implementation, the doctors should confirm the scoring policy, the definition of a scoreable concern, the treatment of single-session and incomplete VALD data, and the sample patient cases used for acceptance testing.
