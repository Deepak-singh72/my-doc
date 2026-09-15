# Customer Agent — Implemented Changes and File Guide

This is a short, current-state list of the work implemented in the backend and
frontend development branches. It is grouped by feature so it can be explained
quickly during a developer review.

## 1. Backend orchestration and interview flow

| Files changed | What we changed |
|---|---|
| `DataHandling/server.py` | Hardened the main REST/WebSocket flow, isolated each connection's agent state, added safe start/resume/new-attempt handling, moved blocking work away from the async event loop, improved persistence, upload handling, clinical escalation, duplicate-request handling, and prompt-leak barriers. |
| `DataHandling/src/graph/graph.py` | Simplified the primary LangGraph topology and removed unnecessary sequential processing. |
| `DataHandling/src/graph/edges.py` | Corrected routing between extraction, validation, uploads, questions, summaries, and corrections. |
| `DataHandling/src/graph/state.py` | Cleaned the shared interview-state contract and removed obsolete correction-routing state. |
| `DataHandling/src/graph/server_adapter.py` | Kept graph results, progress, attachments, form identity, and WebSocket responses synchronized. |
| `DataHandling/src/graph/nodes/extract.py` | Combined extraction and intent work, reduced avoidable AI latency, and removed the incorrect normal-answer correction branch. |
| `DataHandling/src/graph/nodes/first_turn.py` | Improved first-message extraction and routing. |
| `DataHandling/src/graph/nodes/generate.py` | Improved next-question/summary transitions and deterministic referral handling. |
| `DataHandling/src/graph/nodes/correction.py` and `nodes/summary.py` | Restricted correction processing to the summary-confirmation flow and improved summary response handling. |
| `DataHandling/src/graph/pure_functions/form_extraction.py` | Improved structured field extraction, referral extraction, negative answers, corrections, and PROM-safe behavior. |
| `DataHandling/src/graph/pure_functions/intent_detection.py` | Removed broad keywords such as ordinary “sorry/actually/change” usage from active correction routing, preventing repeated correction messages during normal intake. |
| `DataHandling/src/graph/pure_functions/reasoning_extractor.py` and `summary.py` | Improved extraction/summary behavior and reduced unsafe or duplicate processing. |
| `DataHandling/src/llm/functionalities.py` and `src/llm/utils.py` | Reduced the legacy `HealthAgent` implementation, retained it as a fallback, and aligned provider/model handling with the active graph. |

## 2. Form data, ownership and repeat assessments

| Files changed/added | What we changed |
|---|---|
| `DataHandling/app/db/ownership.py` | Added reusable patient/form ownership checks so one patient cannot load another patient's form. |
| `DataHandling/app/db/queries.py` | Centralized ObjectId/string-compatible, user-scoped MongoDB queries and prevented broad form updates/deletes. |
| `DataHandling/app/db/connection.py` and `app/db/mongo.py` | Centralized MongoDB access and enforced TLS certificate/hostname verification. |
| `DataHandling/app/forms/attempts.py` | Added opaque `attemptId` values and indexes so repeated assessments remain separate. |
| `DataHandling/app/forms/lifecycle.py` | Added monotonic `draft`, `in_progress`, and `completed` lifecycle rules; only drafts can expire. |
| `DataHandling/app/forms/progress.py` | Centralized form and section completion calculation. |
| `DataHandling/app/forms/titles.py` | Replaced an unnecessary AI title request with deterministic form titles. |
| `DataHandling/app/forms/instruments.py` and `app/forms/prom.py` | Added stable PROM question identity, administered-question snapshots, structured-answer validation, and safe scoring state. |

## 3. WebSocket, frontend and audio reliability

| Files changed/added | What we changed |
|---|---|
| `DataHandling/app/ws/idempotency.py` | Added bounded request-ID tracking so retries do not save or process the same answer twice. |
| `DataHandling/app/runtime/blocking.py` | Added a bounded thread/executor boundary for synchronous database and provider work. |
| `DataHandling/app/audio/limits.py` | Added server-side recording duration, chunk, and total-byte limits. |
| `DataHandling/app/audio/stt.py` | Centralized the audio model, improved fallback handling, moved the Gemini instruction to `system_instruction`, and rejects prompt-contaminated transcripts before returning them. |
| `DataHandling/app/content_safety.py` | Added deterministic detection and recursive removal of leaked internal transcription instructions. |
| `../customer-agent-frontend/src/hooks/useWebSocket.ts` | Prevented parallel sockets, cleaned timers/buffers on unmount, stopped unrecoverable reconnect loops, handled duplicate acknowledgements, and aligned messages with the backend contract. |
| `../customer-agent-frontend/src/hooks/useVoiceRecorder.ts` | Improved MediaRecorder/audio cleanup and bounded recording behavior. |
| `../customer-agent-frontend/src/components/TranscriptionInterface.tsx` | Prevented auto-send/manual-send duplication, added request IDs, improved voice state cleanup, consent checks, upload behavior, and structured PROM controls. |

## 4. Clinical safety, privacy and observability

| Files changed/added | What we changed |
|---|---|
| `DataHandling/app/clinical/escalation.py` | Added deterministic detection and handling for urgent clinical risk, with a dedicated stop-interview WebSocket response. |
| `DataHandling/app/observability/privacy.py` | Added pseudonymous identifiers and safe exception-type logging to avoid normal logs containing patient answers or credentials. |
| `DataHandling/app/observability/ai_usage.py` | Added provider/model/operation call counts, latency, token/audio usage, failures, and estimated-cost metrics. |
| `DataHandling/app/ai/models.py` | Centralized approved Gemini models, defaults and fallbacks; retired or unapproved model IDs now fail configuration validation. |
| `DataHandling/app/mcp_client.py` | Made the optional MCP integration safer and kept it out of the required production path. |

## 5. Upload and report-processing reliability

| Files changed/added | What we changed |
|---|---|
| `DataHandling/app/uploads/policy.py` | Added file-count, per-file, total-size, signature/MIME, PDF-page, image-pixel, and image-dimension validation. |
| `DataHandling/app/uploads/reading.py` | Added bounded streaming reads so an oversized upload is rejected before consuming unlimited memory. |
| `DataHandling/upload/s3_client.py` | Improved safe S3 upload/download behavior and metadata handling. |
| `DataHandling/app/jobs/reports.py` | Replaced fragile background-only processing with MongoDB-backed jobs, leases, retries, recovery, and stale-result protection. |
| `DataHandling/docscanner/service.py` | Added safe PDF/image inspection, bounded rendering, multi-report limits, and stricter model-output handling. |
| `DataHandling/docscanner/client.py` | Centralized Bedrock configuration, retries and usage monitoring. |

## 6. Frontend configuration, quality and dependency work

| Files changed | What we changed |
|---|---|
| `../customer-agent-frontend/src/config/api.ts`, `config/policy.ts`, `utils/api-config.ts` | Centralized API, WebSocket, consent and environment URL validation; added controlled HTTP/IP support for isolated UAT. |
| `../customer-agent-frontend/vite.config.ts` and `src/vite-env.d.ts` | Added required build-time configuration validation and TypeScript environment definitions. |
| `../customer-agent-frontend/src/App.tsx`, `pages/Index.tsx`, `pages/ConsentPage.tsx`, `pages/FormSelection.tsx` | Improved patient/form URL routing, consent redirect behavior and form selection. |
| `../customer-agent-frontend/src/utils/graphql-client.ts` and data hooks | Removed unsafe defaults and aligned configured GraphQL requests. |
| `../customer-agent-frontend/eslint.config.js` and affected components | Fixed TypeScript/ESLint issues and unsafe loose types. |
| `../customer-agent-frontend/package.json` and `package-lock.json` | Updated vulnerable dependencies and standardized npm lockfile usage. |
| `../customer-agent-frontend/.gitignore`, `.dockerignore`, `.vercelignore` | Prevented local environment files, build output and unrelated artifacts from entering deployments/repository history. |

## 7. Deployment and repository cleanup

| Files changed/added | What we changed |
|---|---|
| `DataHandling/deployment/Dockerfile` | Changed to a multi-stage, allow-list-based runtime image and removed tests, credentials, local data and build tools from the final image. |
| `docker-compose.dev-isolated.yml` | Added an isolated backend service/container, port, volumes and network so UAT does not replace the existing production container. |
| `../customer-agent-frontend/Dockerfile.dev`, `docker-compose.dev-isolated.yml`, `nginx.dev.conf` | Added an isolated production-style frontend build served by Nginx on its own UAT port. |
| `DataHandling/.env.example` and frontend `.env.example` | Documented required configuration names without committing real secrets. |
| Root/DataHandling `.gitignore` and `.dockerignore` | Excluded credentials, generated audio/transcripts, reports, caches, local databases and build artifacts. |
| `DataHandling/deployment/update.sh`, `update-fast.sh`, the temporary token generator, stale output files and unused frontend assets | Removed unsafe, generated or obsolete repository/deployment artifacts. |
| `DataHandling/requirements.in`, `requirements-docker.txt`, `requirements.txt` | Separated direct dependencies and pinned the reproducible container dependency set. |

## 8. Documentation and tests

| Files changed/added | What we changed |
|---|---|
| `README.md` and `DataHandling/docs/README.md` | Replaced outdated architecture descriptions with the current application boundaries. |
| `DataHandling/docs/PUBLIC_API.md` | Documented current REST/WebSocket messages, form identity and persistence behavior. |
| `DataHandling/docs/LOCAL_DEV.md` | Added safe local configuration, startup and verification steps. |
| `DataHandling/docs/DEPLOYMENT.md` and `DEV_ISOLATED_DEPLOYMENT.md` | Added deployment checks and isolated shared-server UAT instructions. |
| `DataHandling/tests/test_*.py` | Added regression coverage for ownership, query scope, attempts, lifecycle, PROMs, uploads, report jobs, WebSockets, async boundaries, clinical escalation, privacy, model configuration, transcription prompt leakage and interview correction routing. |
| Frontend build/lint/audit workflow | Verified TypeScript compilation, ESLint, production build and npm dependency audit after the frontend changes. |

## 9. Before vs now — code evidence

The **Before** column is taken from the code immediately before the main
hardening work (`d7db465^`). The **Now** column shows the active development
implementation. The snippets are intentionally short so they can be searched
and opened quickly during a screen-sharing review.

| Area and file | Before | Now | Why the current implementation is needed |
|---|---|---|---|
| Form deletion — `DataHandling/server.py` | `delete_one({"formId": existing_form_id})` | `delete_filter = build_owned_form_filter(existing_form, user_id_for_check, existing_form_id)` then `delete_one(delete_filter)` | The old filter was not patient-scoped. The current filter requires the exact document ID, stored patient ID and form ID before deletion. |
| Form lookup — `DataHandling/server.py`, `app/db/queries.py` | `find_one({"formId": form_id, "userId": _uid})` | `build_user_form_queries(user_id, form_id, attempt_id=attempt_id)` | Centralizes ObjectId/string handling and ensures reads stay inside the requested patient's form/attempt. |
| Repeat assessments — `server.py`, `app/forms/attempts.py` | Unique identity was only `("userId", "formId")`. | `ATTEMPT_INDEX_KEYS = [("userId", 1), ("formId", 1), ("attemptId", 1)]` | A new assessment now has its own opaque attempt identity instead of overwriting the patient's earlier submission. |
| Form writes — `DataHandling/server.py` | `{"formId": form_id, "userId": _uid}` | `add_attempt_write_scope({"formId": form_id, "userId": _uid}, attempt_id)` | Updates target one assessment attempt, not every record that shares the form template ID. |
| Form lifecycle/TTL — `app/db/mongo.py`, `app/forms/lifecycle.py` | TTL was tied to `{"title": "New Form"}` and `createdAt`. | TTL uses `expiresAt` with `partialFilterExpression={"status": DRAFT}`; completed status returns `expires_at=None`. | A title is presentation data, not lifecycle state. Only abandoned drafts can now expire; completed clinical records cannot be deleted by draft cleanup. |
| Form title cost — `DataHandling/server.py`, `app/forms/titles.py` | `form_title = generate_form_title(form_data_copy)` called the LLM. | `form_title = build_form_title(form_data_copy, empty_title="New Form")` | A deterministic title does not need an AI request, so it reduces latency, cost and failure points. |
| Normal-answer routing — `src/graph/nodes/extract.py`, `src/graph/edges.py` | `should_check_for_correction(user_input, awaiting_confirmation=False)` could route words such as “actually” into correction mode. | `if intent == "request_change" and correction_text: return "detect_correction"` after summary confirmation. | Correction handling now runs only in the correction context, preventing repeated “I couldn't identify the exact change” responses during normal intake. |
| Audio prompt handling — `app/audio/stt.py` | `contents=[audio_part, _GEMINI_STT_PROMPT]` sent the internal instruction as ordinary content. | `contents=[audio_part]` with `GenerateContentConfig(system_instruction=_GEMINI_STT_PROMPT, ...)`. | Keeps system instructions separate from patient audio and materially reduces the chance that the model returns the internal prompt as transcript text. |
| Prompt-leak barrier — `app/audio/stt.py`, `app/content_safety.py`, `server.py` | `return text` accepted the provider transcript directly. | `contains_internal_transcription_prompt(text)` triggers fallback/rejection, and `scrub_internal_transcription_prompts(...)` protects persistence. | A leaked instruction is rejected before display and scrubbed before any form/chat value can enter the clinical record. |
| AI model selection — `app/audio/stt.py`, `app/ai/models.py` | `model="gemini-2.5-flash"` and other model IDs were spread across modules. | `model=MODEL_REGISTRY.audio`; `MODEL_REGISTRY = build_model_registry(os.environ)` rejects retired/unapproved IDs during import. | One validated registry controls model changes and prevents silent use of inconsistent or retired models. |
| AI monitoring — provider call sites and `app/observability/ai_usage.py` | Provider methods were called directly, with ad-hoc timing logs. | `tracked_ai_call(provider=..., model=..., operation=..., call=..., usage_extractor=...)` | Records call count, errors, latency, usage and estimated cost using one consistent boundary. |
| Async database/provider work — `DataHandling/server.py`, `app/runtime/blocking.py` | `asyncio.create_task(asyncio.to_thread(save_customer_info, ...))` was fire-and-forget and used the default executor. | `await run_blocking(save_customer_info, ...)` uses the bounded application executor. | The save completes before success continues, exceptions are observable, and synchronous work cannot create unlimited default-executor pressure. |
| Duplicate submissions — frontend `useWebSocket.ts`, `TranscriptionInterface.tsx`; backend `app/ws/idempotency.py` | `sendTextInput(text)` had no request identity. | Frontend sends `requestId`; backend calls `_request_window.register(user_id, requestId, questionId)` and returns `submission_ack` for duplicates. | A retry, reconnect or UI race cannot process and persist the same patient answer twice. |
| WebSocket policy failures — frontend `src/hooks/useWebSocket.ts` | Every non-`1000` close retried after three seconds. | `if (event.code === 4003 \|\| event.code === 1008) shouldReconnectRef.current = false`. | Clinical-stop and policy/authentication failures cannot be fixed by reconnecting, so the current code prevents an endless reconnect loop. |
| Upload memory safety — `DataHandling/server.py`, `app/uploads/reading.py` | `file_bytes = await file.read()` loaded an unlimited request file into memory. | `file_bytes = await read_upload_bounded(file, max_bytes=read_limit)` followed by `inspect_upload(...)`. | Oversized or invalid input is stopped while streaming, before it can consume unbounded memory or reach S3/document processing. |
| Report durability — `DataHandling/server.py`, `app/jobs/reports.py` | `asyncio.create_task(_run_ocr_and_update())` existed only inside the API process. | `job = build_report_job(...)` followed by `report_jobs_collection.insert_one(job)`; workers claim jobs using leases and retries. | Report work survives request completion and process restarts, and stale workers cannot overwrite newer results. |
| Clinical escalation — `server.py`, `app/clinical/escalation.py` | There was no deterministic pre-AI urgent-risk gate. | `_escalation = assess_urgent_risk(text_input, ...)` sends `clinical_escalation` and closes with code `4003`. | Urgent warning signs take a fixed safety route before normal AI interview processing continues. |
| MongoDB TLS — `app/db/connection.py`, `app/db/mongo.py` | Modules created `MongoClient(...)` independently. | `validate_mongo_tls_options(uri)` then `MongoClient(uri, **build_verified_mongo_options(...))`. | Connection creation is centralized and refuses options that disable certificate or hostname verification. |
| Frontend endpoint configuration — frontend `src/config/api.ts`, `vite.config.ts` | Runtime URLs/defaults were resolved in multiple places. | `resolveRuntimeEndpoints(...)` validates the required environment and API/WS/consent URL policy at build/startup. | Misconfigured deployments fail visibly instead of building a frontend that connects to the wrong service. |

### How to demonstrate one item in a review

Use the **Area and file** column to open the current implementation, search for
the exact expression shown in **Now**, and explain the final column. If Git
history is requested, the baseline can be displayed with:

```bash
git show d7db465^:DataHandling/server.py
```

For frontend history, use the commit before its hardening change:

```bash
git -C ../customer-agent-frontend show e3cde01^:src/hooks/useWebSocket.ts
```

These commands are review references only; they do not modify the working tree.

## 10. Important current-state clarifications

- A separate customer-agent access-token system was initially implemented, but
  it was intentionally removed after the product owner confirmed that the
  existing OTP/consent integration must remain the active flow. Do not present
  patient-link tokens as a current feature.
- Direct public HTTP/IP support was added only for isolated development/UAT. It
  is not the recommended production exposure model; production should use HTTPS,
  WSS and the approved upstream boundary.
- `context-layer/` was added by the earlier developer and later merged into this
  branch. We reviewed and documented how it connects, but it should not be
  presented as part of our optimization implementation.

## 11. Short meeting summary

> We hardened the complete patient-intake path across frontend, WebSocket,
> LangGraph, MongoDB and external AI services. The main results are isolated
> patient sessions, correct form ownership and repeat-attempt storage, fewer and
> safer AI calls, reliable audio/report processing, privacy-safe monitoring,
> prompt-leak and clinical-safety protection, cleaned deployment artifacts, and
> regression coverage for the critical flows. We also added isolated backend and
> frontend Docker deployment files for UAT without replacing production.
