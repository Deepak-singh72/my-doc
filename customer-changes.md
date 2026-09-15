# Customer Agent — Implementation Explanation Guide

This document explains the implemented work in simple Hinglish. Each item gives
the reason, exact files, current code reference, and a short explanation that can
be used during a client or developer review.

## How to use this document

For any review question, answer in this order:

1. **Problem:** Pehle kya risk/problem tha?
2. **Implementation:** Humne technically kya change kiya?
3. **Location:** Change kis file/function mein hai?
4. **Result:** Isse security, reliability, performance ya cost par kya effect hua?

Items **1–26** are the Customer Agent hardening and optimization work. Item
**27 (`context-layer`) was implemented by the earlier developer**; we reviewed
its integration and documented its role.

---

## 1. Backend architecture refactor and organization

**What and why:** Large `server.py` responsibilities ko small, testable modules
mein separate kiya. Isse main WebSocket orchestration readable raha aur database,
forms, uploads, AI, safety and job logic independently test ho sakti hai.

**Main files:**

- `DataHandling/server.py` — REST/WebSocket entry and orchestration
- `DataHandling/src/graph/graph.py` — LangGraph workflow
- `DataHandling/src/graph/edges.py` — workflow routing
- `DataHandling/src/graph/nodes/` — individual interview operations
- `DataHandling/app/` — database, forms, audio, uploads, jobs and safety modules

**Code to show:**

```python
_interview_graph = build_interview_graph(...)
graph_state = build_graph_state(client_state, text_input, _save_for_graph)
result_state = await run_blocking(_interview_graph.invoke, graph_state)
```

**Meeting explanation:** “`server.py` transport/orchestration handle karta hai;
business rules reusable `app/` modules aur interview workflow `src/graph/` mein
separate hai.”

## 2. Separate form attempt IDs

**What and why:** Same patient ke repeated FRM-01/FRM-02 submissions ko separate
records banane ke liye opaque `attemptId` add kiya, taaki previous assessment
overwrite na ho.

**Files:** `app/forms/attempts.py`, `server.py`

```python
ATTEMPT_INDEX_KEYS = [
    ("userId", 1), ("formId", 1), ("attemptId", 1)
]
attempt_id = new_form_attempt_id()
```

**Meeting explanation:** “`formId` questionnaire ko identify karta hai aur
`attemptId` us questionnaire ki individual submission ko.”

## 3. Form lifecycle states

**What and why:** Form ki actual state track karne ke liye explicit lifecycle
add ki: `draft`, `in_progress`, and `completed`.

**Files:** `app/forms/lifecycle.py`, `server.py`

```python
DRAFT = "draft"
IN_PROGRESS = "in_progress"
COMPLETED = "completed"
lifecycle = resolve_form_lifecycle(...)
```

**Meeting explanation:** “Status title ya incomplete data se guess nahi hota;
central lifecycle function se consistently calculate hota hai.”

## 4. Completed forms protected from cleanup

**What and why:** Old cleanup title/creation date par dependent tha. Ab only
expired drafts ko TTL delete kar sakta hai; completed record ka `expiresAt`
remove hota hai.

**Files:** `app/forms/lifecycle.py`, `app/db/mongo.py`

```python
collection.create_index(
    [("expiresAt", 1)],
    expireAfterSeconds=0,
    partialFilterExpression={"status": DRAFT},
)
```

**Meeting explanation:** “Completed clinical record TTL condition match hi nahi
karta, isliye draft cleanup usko remove nahi kar sakta.”

## 5. Correct patient form update/delete queries

**What and why:** Broad `formId` query ki jagah exact patient-owned document
filter banaya, taaki another patient ka record update/delete na ho.

**Files:** `app/db/ownership.py`, `app/db/queries.py`, `server.py`

```python
delete_filter = build_owned_form_filter(
    existing_form, user_id_for_check, existing_form_id
)
await run_blocking(customer_info_collection.delete_one, delete_filter)
```

```python
queries = build_user_form_queries(
    user_id, form_id, attempt_id=attempt_id
)
```

**Meeting explanation:** “Mutation se pehle stored document ownership verify
hoti hai, and query patient + form + optional attempt tak scoped rehti hai.”

## 6. Duplicate answers and reconnect handling

**What and why:** Network retry, double click ya auto/manual send race se same
answer twice process ho sakta tha. Frontend request ID bhejta hai and backend
bounded idempotency window duplicate reject karta hai.

**Files:** `app/ws/idempotency.py`, frontend `src/hooks/useWebSocket.ts`, frontend
`src/components/TranscriptionInterface.tsx`

```python
_request_decision = _request_window.register(
    client_state.get("user_id"),
    data.get("requestId"),
    data.get("questionId"),
)
if _request_decision is RequestDecision.DUPLICATE:
    await websocket.send_text(json.dumps({
        "type": "submission_ack",
        "status": "duplicate",
        "requestId": data.get("requestId"),
    }))
```

```typescript
sendTextInput(text, { requestId, questionId });
```

**Meeting explanation:** “Ek logical answer ka ek `requestId` hai, so resend ko
acknowledge kiya jata hai but dobara extract/save nahi kiya jata.”

## 7. Per-connection patient session state

**What and why:** Shared mutable agent state concurrent patients ka data mix kar
sakta tha. Ab `client_state` and active agent WebSocket handler ke andar create
hote hain.

**File:** `server.py`

```python
@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    health_agent = await run_blocking(HealthAgent)
    client_state = {
        "user_id": None,
        "form_id": None,
        "attempt_id": None,
        "received_audio_buffer": bytearray(),
        # remaining graph/session fields
    }
```

**Meeting explanation:** “Har socket ka independent interview form, phase,
history, attempt and audio buffer hai.”

## 8. Microphone duration and audio-size limits

**What and why:** Unlimited recording server memory and STT cost exhaust kar
sakti thi. Server recording duration, chunk size and accumulated byte count
validate karta hai.

**Files:** `app/audio/limits.py`, `server.py`, frontend
`src/hooks/useVoiceRecorder.ts`

```python
limit_error = audio_limit_error(
    current_bytes=len(received_audio_buffer),
    incoming_bytes=len(audio_chunk),
    elapsed_seconds=elapsed,
    max_total_bytes=MAX_AUDIO_SESSION_BYTES,
    max_duration_seconds=MAX_AUDIO_SESSION_SECONDS,
)
```

**Meeting explanation:** “Frontend user experience ke liye recording stop karta
hai; backend authoritative limits enforce karta hai.”

## 9. Secure PDF/image upload validation

**What and why:** Extension alone reliable nahi hoti. Streaming size, signature,
MIME, total quota, PDF pages, image dimensions and pixel count validate kiye.

**Files:** `app/uploads/reading.py`, `app/uploads/policy.py`, `server.py`

```python
file_bytes = await read_upload_bounded(file, max_bytes=read_limit)
validate_upload_quotas(...)
inspection = await run_blocking(
    inspect_report_upload,
    file_bytes,
    filename,
    file.content_type,
)
```

`docscanner/service.py` ke andar `inspect_report_upload()`
`validate_upload_identity()` and `validate_document_complexity()` call karta hai.

**Meeting explanation:** “Invalid/oversized content S3 ya document AI tak pahunchne
se pehle reject hota hai.”

## 10. Durable report processing

**What and why:** In-process background task server restart par lost ho sakti thi.
Report jobs MongoDB mein persist hote hain and worker leases, retries and recovery
use karta hai.

**Files:** `app/jobs/reports.py`, `server.py`, `docscanner/service.py`

```python
job = build_report_job(...)
await run_blocking(report_jobs_collection.insert_one, job)
job = claim_report_job(report_jobs_collection, worker_id, ...)
```

**Meeting explanation:** “API response ke baad bhi job database mein rahti hai;
failed/restarted worker ke baad another worker safely retry kar sakta hai.”

## 11. Central Gemini model configuration

**What and why:** Scattered/hardcoded model IDs ko one validated registry se
replace kiya. Retired or unapproved models startup configuration mein fail hote
hain.

**File:** `app/ai/models.py`; consumers include `app/audio/stt.py`, `server.py`
and `src/llm/`.

```python
MODEL_REGISTRY = build_model_registry(os.environ)
model = MODEL_REGISTRY.audio
```

**Meeting explanation:** “Model upgrade ek reviewed registry change hai, random
file-level string replacement nahi.”

## 12. Unnecessary AI calls reduced

**What and why:** Deterministic tasks ko LLM se remove kiya, especially form
title creation and structured PROM paths. Isse latency, cost and nondeterminism
reduce hua.

**Files:** `app/forms/titles.py`, `app/forms/prom.py`, `server.py`,
`src/graph/nodes/extract.py`

```python
form_title = build_form_title(form_data_copy, empty_title="New Form")
structured_answers = parse_structured_prom_answers(
    text_input, question_meta, data.get("inputMode")
)
```

**Meeting explanation:** “AI sirf language/reasoning wale work ke liye use hota
hai; simple title and already-structured answers deterministic code handle karta
hai.”

## 13. AI usage, latency and estimated-cost monitoring

**What and why:** Provider usage measurable banane ke liye every important AI
boundary par common tracking wrapper add kiya.

**File:** `app/observability/ai_usage.py`; used by audio, LLM and document clients.

```python
response = tracked_ai_call(
    provider="google_genai",
    model=MODEL_REGISTRY.audio,
    operation="audio_transcription",
    call=provider_call,
    usage_extractor=gemini_usage,
)
```

**Meeting explanation:** “Per provider/model/operation calls, failures, latency,
usage and configured-price-based estimated cost record hota hai.”

## 14. Patient information removed from standard logs

**What and why:** Raw user IDs, answers and exception strings normal logs mein
PHI/secrets expose kar sakte the. Pseudonymous ID and exception type helpers add
kiye.

**File:** `app/observability/privacy.py` and backend logging call sites.

```python
_log.warning(
    "internal_prompt_content_rejected",
    subject=pseudonymous_id(user_id),
)
_log.error("operation_failed", error_type=error_type(exc))
```

**Meeting explanation:** “Operational event visible rehta hai, patient answer aur
credential log nahi hota.”

## 15. Deterministic clinical emergency escalation

**What and why:** Urgent-risk messages normal generative interview se pass nahi
hone chahiye. LLM calls se before deterministic gate add kiya and decision audit
record persist hota hai.

**Files:** `app/clinical/escalation.py`, `server.py`

```python
_escalation = assess_urgent_risk(text_input, ...)
if _escalation is not None:
    await websocket.send_text(clinical_escalation_message)
    await websocket.close(code=4003)
```

**Meeting explanation:** “Known urgent signals fixed safety policy follow karte
hain and normal interview immediately stop hota hai.”

## 16. MongoDB TLS certificate validation

**What and why:** Mongo connections centralized kiye and certificate/hostname
verification disable karne wale URI options reject kiye.

**Files:** `app/db/connection.py`, `app/db/mongo.py`, `server.py`

```python
validate_mongo_tls_options(uri)
client = MongoClient(uri, **build_verified_mongo_options(...))
```

**Meeting explanation:** “Application insecure TLS override ke saath database
connection start nahi karti.”

## 17. PROM identity, metadata and structured answers

**What and why:** PROM scoring ke liye question identity and administered version
stable rehna zaroori hai. Snapshot, definition hash and structured answer
validation add ki.

**Files:** `app/forms/instruments.py`, `app/forms/prom.py`, `server.py`

```python
snapshot = build_prom_snapshot(
    questions,
    source_form_id=provided_form_id,
    legacy_form_data=existing_prom_data,
)
answers = parse_structured_prom_answers(
    text_input, question_meta, data.get("inputMode")
)
updated_snapshot = apply_prom_answers(snapshot, answers_by_question_id)
```

**Meeting explanation:** “Hum sirf current question bank par depend nahi karte;
patient ko actually administered questions ka immutable snapshot save hota hai.”

## 18. Repeated questions and correction detection

**What and why:** Normal statements containing “actually”, “sorry” or “change”
incorrectly correction route mein ja rahe the. Correction routing summary
confirmation context tak restrict ki.

**Files:** `src/graph/edges.py`, `src/graph/nodes/extract.py`,
`src/graph/nodes/correction.py`, `src/graph/nodes/summary.py`,
`src/graph/pure_functions/intent_detection.py`

```python
if intent == "request_change":
    if correction_text:
        return "detect_correction"
```

**Meeting explanation:** “Normal intake answer extraction mein correction keyword
heuristic active nahi hai; explicit summary correction par hi correction node run
hota hai.”

## 19. Transcription system-prompt leakage barriers

**What and why:** STT system instruction transcript ke saath return ho kar patient
chat/clinical record mein save ho sakta tha. Prompt separation, detection,
fallback/rejection and persistence scrubbing add ki.

**Files:** `app/audio/stt.py`, `app/content_safety.py`, `server.py`

```python
config=GenerateContentConfig(
    system_instruction=_GEMINI_STT_PROMPT,
    temperature=0,
)
```

```python
if contains_internal_transcription_prompt(text):
    raise ValueError("Transcription rejected by internal-content guard")

clean_form, removed = scrub_internal_transcription_prompts(form)
```

**Meeting explanation:** “Provider output trusted data nahi hai; display and save
se pehle internal-instruction signatures reject/scrub hoti hain.”

## 20. Frontend WebSocket cleanup and duplicate prevention

**What and why:** Unmounted screen ke sockets, timers and audio buffers resource
leak/reconnect loop create kar rahe the. Mounted/socket identity checks and full
cleanup add kiya.

**Files:** frontend `src/hooks/useWebSocket.ts`,
`src/components/TranscriptionInterface.tsx`, `src/hooks/useVoiceRecorder.ts`

```typescript
if (!isMountedRef.current || wsRef.current !== ws) return;
if (event.code === 4003 || event.code === 1008) {
  shouldReconnectRef.current = false;
}
```

```typescript
return () => {
  isMountedRef.current = false;
  clearTimeout(reconnectTimeoutRef.current);
  audioBuffers.clear();
  ws.close(1000, "Component unmounted");
};
```

**Meeting explanation:** “Only current mounted component socket state update karta
hai; permanent policy errors retry loop mein nahi jaate.”

## 21. Frontend TypeScript, ESLint and dependency fixes

**What and why:** Loose types, lint failures and vulnerable dependency versions
resolve kiye so CI/build repeatable and dependency audit clean rahe.

**Files:** frontend `eslint.config.js`, `package.json`, `package-lock.json`, and
affected hooks/components.

```bash
npm run lint
npm run build
npm audit
```

**Meeting explanation:** “Frontend type-check/lint/build successfully validate
kiya and both relevant npm audits zero known vulnerabilities report karte the at
verification time.”

## 22. Isolated backend and frontend Docker configuration

**What and why:** Same server par UAT deploy ko existing prod/dev containers,
ports and network se separate rakha.

**Files:**

- Backend: `docker-compose.dev-isolated.yml`
- Backend image: `DataHandling/deployment/Dockerfile`
- Frontend: `../customer-agent-frontend/docker-compose.dev-isolated.yml`
- Frontend image: `../customer-agent-frontend/Dockerfile.dev`
- Frontend Nginx: `../customer-agent-frontend/nginx.dev.conf`

```yaml
services:
  customer-agent-dev-isolated:
    ports:
      - "0.0.0.0:8004:8000"
```

**Meeting explanation:** “UAT ka project name, container, host port and service
configuration separate hai, so production container replace nahi hota.”

## 23. Backend Docker image size reduction

**What and why:** Multi-stage and allow-list runtime build se tests, credentials,
local data, caches and build tools final image se remove kiye.

**Files:** `DataHandling/deployment/Dockerfile`, `.dockerignore`

```dockerfile
FROM python:3.12-slim AS builder
# build dependencies and wheels

FROM python:3.12-slim AS runtime
# copy only runtime dependencies and allow-listed application paths
```

**Result:** Measured image approximately **575 MB se 428 MB** hui.

**Meeting explanation:** “Final container mein application run karne ke required
artifacts hi hain, complete repository/build toolchain nahi.”

## 24. Repository cleanup

**What and why:** Generated audio/reports, caches, environment files, obsolete
deployment scripts and unused artifacts Git/deployment scope se remove/exclude
kiye.

**Files:** root and `DataHandling` `.gitignore`/`.dockerignore`; frontend
`.gitignore`, `.dockerignore`, `.vercelignore`.

Removed examples include old `deployment/update.sh`, `update-fast.sh`, temporary
token generator, stale outputs and unused frontend assets.

**Meeting explanation:** “Repository mein source, reviewed configuration and
documentation rehte hain; secrets/generated runtime output nahi.”

## 25. Local, API, deployment and project-flow documentation

**What and why:** Setup and runtime knowledge scattered tha. Current configuration,
interfaces and operational steps explicit documents mein add kiye.

**Files:**

- `README.md`
- `DataHandling/docs/README.md`
- `DataHandling/docs/LOCAL_DEV.md`
- `DataHandling/docs/PUBLIC_API.md`
- `DataHandling/docs/DEPLOYMENT.md`
- `DataHandling/docs/DEV_ISOLATED_DEPLOYMENT.md`
- `DataHandling/docs/PROJECT_CHANGES_IMPLEMENTED.md`
- `DataHandling/docs/IMPLEMENTATION_EXPLANATION_GUIDE.md`

**Meeting explanation:** “Developer setup, REST/WebSocket contract, architecture
and deployment procedure documented and reviewable hain.”

## 26. Comprehensive regression tests

**What and why:** Critical fixes ko future regression se protect karne ke liye
focused unit/contract/source-boundary tests add kiye.

**Folder:** `DataHandling/tests/`

Important tests:

- `test_db_ownership.py`, `test_db_queries.py`
- `test_form_attempts.py`, `test_form_lifecycle.py`
- `test_audio_limits.py`, `test_transcription_content_safety.py`
- `test_upload_policy.py`, `test_report_jobs.py`
- `test_prom_instruments.py`, `test_prom_structured.py`
- `test_clinical_escalation.py`, `test_clinical_escalation_boundary.py`
- `test_ws_idempotency.py`, `test_websocket_audio_auth_regression.py`
- `test_ai_models.py`, `test_ai_usage.py`
- `test_async_boundaries.py`, `test_container_contract.py`
- `test_interview_correction_routing.py`

```bash
cd DataHandling
pytest -q
```

**Meeting explanation:** “Tests individual helper behavior ke saath critical
integration boundaries bhi assert karte hain, for example escalation must execute
before AI and unsafe transcript must not reach persistence.”

## 27. Context-layer assessment-test recommender

> **Ownership clarification:** `context-layer/` earlier developer ne implement
> kiya tha. Humne iska role and integration review/document kiya; ise apna original
> optimization implementation present nahi karna hai.

**Purpose:** Completed customer intake and clinical context ke basis par clinician
assessment form ke liye relevant objective tests recommend karta hai.

**Backend files:** `context-layer/` (separate service/container, normally port
`8002`).

**Dashboard integration:**

- `stance-dashboard-frontend/src/app/(protected)/patients/[id]/timeline/page.tsx`
- `stance-dashboard-frontend/src/utils/agentFormDataMapper.ts`
- `stance-dashboard-frontend/next.config.ts`

```typescript
const params = new URLSearchParams({ userId, appointmentId, reportId });
const response = await fetch(`/recommendations?${params}`);
const assessmentData = mapAgentFormDataToAssessment(await response.json());
```

**Runtime flow:**

```text
Dashboard patient timeline
  → GET /recommendations
  → Next.js proxy
  → context-layer service
  → patient-derived recommended tests
  → agentFormDataMapper
  → NewReportForm objective-assessment section
```

**Meeting explanation:** “Context layer patient se questions ask nahi karta. It
reads available context and recommends tests to the clinician. Question asking
and answer persistence remain Customer Agent responsibilities.”

---

## Dashboard connection in one minute

```text
Dashboard creates/copies /{patientId}/FRM-01 link
  → patient uses customer-agent-frontend
  → frontend talks to DataHandling/server.py over WebSocket
  → backend saves stance-dashboard.customer-info
  → main GraphQL API exposes customerInfo/customerInfoList
  → stance-dashboard-frontend displays the submitted form
```

For FRM-02:

```text
Dashboard Clinical Admin assigns/publishes questions
  → Clinical API writes tagged-questions
  → Customer Agent reads tagged-questions
  → patient completes FRM-02
  → result is saved to customer-info
  → Dashboard reads it through GraphQL
```

The dashboard does **not** use the Customer Agent interview WebSocket directly.
The patient-facing frontend uses that WebSocket; the dashboard reads saved intake
results through GraphQL.

## Final short summary for a meeting

> “We hardened the complete patient intake path from browser and WebSocket to
> LangGraph, AI providers and MongoDB. We added isolated session state, exact
> patient/form/attempt ownership, lifecycle protection, duplicate submission
> control, bounded audio/uploads, durable report jobs, centralized AI models and
> usage tracking, privacy-safe logs, deterministic clinical escalation and prompt
> leak protection. We also improved the frontend and deployment boundaries and
> added regression coverage. The separately contributed context-layer service
> recommends assessment tests and is consumed by the clinician dashboard.”
