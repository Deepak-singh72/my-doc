# Client File Change Guide

Use this document when a client asks which part of the backend changed and why.

| File | What changed | Why it matters |
|---|---|---|
| `LLM/shared/concern_scoring.py` | Added one backend formula for concern scores and risk bands. | Scores are calculated consistently instead of trusting AI text alone. |
| `LLM/report-gen/push_report_to_mongo.py` | Validates concern components, score relationships, evidence dates, and recalculates the final score before saving. | Invalid, contradictory, or unsupported concerns are rejected. |
| `LLM/report-gen/run_audit_by_stance_id.py` | Passes the patient's valid timeline dates into output validation and supports a no-write dry-run. | A concern's evidence must relate to data actually loaded for that patient; testing can be done safely. |
| `LLM/report-gen/eba_agent.py` | Tightened the main clinical-AI instructions: limited active concerns, required dated evidence, and clear score components. Added explicit Vertex AI configuration and model fallback. | The AI receives clearer rules and the service is more reliable when a configured model is unavailable. |
| `LLM/report-gen/concise_agent.py` | Added explicit Vertex AI configuration, model fallback, and output limits. | The concise-summary step has predictable model selection and request limits. |
| `LLM/shared/fetch_and_transform_vald.py` | Sanitizes zero-force VALD readings while keeping valid readings from the same session. | Bad readings are less likely to distort force or asymmetry analysis. |
| `LLM/shared/chronological_data_processor.py` | Uses transformed VALD session dates when building the patient timeline. | Clinical and performance evidence is placed on the correct date. |
| `api/phase_analysis_ws.py` | Fixed the phase-service health endpoint. | Local and deployed service monitoring works correctly. |
| `services/clinical-auditor/requirements.txt` and `LLM/requirements.txt` | Aligned the Google AI client dependency for repeatable testing. | The production image needs one tested dependency version. Final deployment validation remains required. |
| `tests/test_concern_scoring.py` | Added tests for score formula, ranges, risk bands, and invalid combinations. | Confirms score validation behaves as intended. |
| `tests/test_concern_persistence.py` | Added tests for evidence and persistence rules. | Confirms unsafe AI concern output is not saved. |
| `tests/test_vald_sanitization.py` | Added tests for zero-force cleanup. | Confirms valid measurements are retained and invalid zeros removed. |
| `tests/test_vald_timeline_dates.py` | Added tests for VALD timeline dates. | Confirms generated timelines use the correct session date. |

## Simple client explanation

> The changes are concentrated in the AI-generation, validation, and VALD-data processing layers. We kept the database output structure used by the dashboard unchanged. The backend now checks that concern scores are calculated consistently, linked to dated patient evidence, and based on safer performance-data handling.

## If the client asks for technical detail

1. The AI proposes a concern and its score components.
2. The backend validates the components and calculates the displayed score itself.
3. The backend requires evidence dated from the patient's loaded timeline.
4. Only validated output can be stored in the existing summary collection.
5. The dashboard reads the same collection and fields as before.

## Status

| Area | Status |
|---|---|
| Backend validation changes | Complete |
| Automated checks | Passed |
| Local service health | Passed |
| Clinician review of new generated samples | Pending |
| Production regeneration of existing summaries | Pending approval |
