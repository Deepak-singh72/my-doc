# Clinician Agent PR — code change review guide

This document is for explaining the optimization PR during a client code review.
It focuses on the files that changed behavior and groups tests/configuration where
the reason is the same.

Comparison used:

- Frontend: development compared with origin/main
- Backend: development compared with origin/prod

## Frontend changes

### Runtime and UI files

| File | What was implemented | Code-level difference | Why it changed |
|---|---|---|---|
| src/App.tsx | Lazy loading for the form page | FormPage is loaded with React lazy and Suspense instead of entering the initial bundle directly | Makes the first screen smaller and faster |
| src/pages/form-page.tsx | Reset coordination and current form integration | Passes resetVersion, current state and recorder reset behavior through the active page | Ensures visible fields, internal renderer state and voice state stay synchronized |
| src/schemas/form-schemas.ts | Shared form definitions | Defines the field structure used by the generic renderer for First Assessment and Assessment | One schema-driven renderer can support multiple forms |
| src/components/forms/form-renderer.tsx | Current generic form renderer | Exposes renderer methods, accepts resetVersion and applies schema-driven changes | Replaces form-specific duplicated renderers |
| src/components/sections/form-section.tsx | Active form section wrapper | Connects FormPage with FormRenderer and its ref | Provides one active section path for all form types |
| src/handlers/form-renderer.handlers.ts | Consistent array/edit actions | Aligns add, remove and field action shapes with reducer types | Prevents nested row updates from using inconsistent payloads |
| src/hooks/use-form-renderer.ts | Current-state handling | Uses the updated actions and exposes current rendered form data | Supports reset, manual edits and AI snapshot comparison |
| src/types/form-renderer.types.ts | Shared action/ref types | Updated action fields and renderer ref contract | Keeps handlers, hooks and components type-safe |
| src/hooks/use-voice-recorder.tsx | Manual-edit and stale-AI protection | Stores the form snapshot sent with a request and rejects an old result if the user edited meanwhile | Prevents late AI output from overwriting clinician changes |
| src/hooks/use-web-socket.ts | Reliable WebSocket lifecycle | Adds request IDs, response correlation, processing timeout, tracked heartbeat/reconnect cleanup and a single normalized callback | Prevents mixed responses, endless busy state and accidental reconnect loops |
| src/utils/auto-submit-manager.ts | Correct auto-save scheduling | Uses the configured delay, cancels old timers and reads the latest form state | Prevents stale or duplicate automatic saves |
| src/handlers/form-submit-handlers.ts | Shared submit route | Sends renderer saves through the common submission helper and cancels pending auto-save when needed | Keeps manual and automatic saves consistent |
| src/utils/form-submission.ts | One submission workflow | Validates the request, calls the common API save and shows success only after confirmation | Avoids different save behavior in different buttons |
| src/utils/report-payload.ts | Strict GraphQL serialization | Preserves zero/null/clear values, converts First Assessment shapes, maps approved aliases and removes unsupported fields | Prevents GraphQL input errors and data loss |
| src/utils/graphql-client.ts | Confirmed and ordered report saves | Uses the shared payload builder, queues saves by appointment and requires a returned report ID | Reduces same-browser save races and false success messages |
| src/utils/api.ts | Smaller shared API surface and report reads | Uses the shared GraphQL client, supports form-aware report creation, existing First Assessment reads and appointment record reads | Avoids duplicate transport/save implementations |
| src/utils/first-assessment.ts | API-to-renderer conversion | Converts objectiveAssessments list into the frontend objectiveAssessment object and rejects multiple groups | Makes reload and save shapes reversible |
| src/utils/records-to-form.ts | Existing clinical-record prefill | Maps Report.records into First Assessment or follow-up Assessment fields | Allows new working reports to start from existing visit records |
| src/hooks/use-form-management.tsx | Safe initialize, reset and save flow | Loads an existing First Assessment before creating one, ignores stale navigation responses, resets from schema defaults and offers guarded Undo | Fixes prefetched values returning after reset and avoids duplicate initialization |
| src/utils/api-config.ts | Environment-only API key | Removes the hardcoded active API-key fallback | Keeps active credentials out of source code |
| vite.config.ts | Environment-driven proxy key | Reads proxy credentials from Vite environment configuration | Removes committed credentials and keeps environments configurable |
| proxy-server.js | Environment-driven proxy key | Reads the API key from process environment instead of source | Same configuration and credential reason |

### Frontend Docker and deployment files

| File | What was implemented | Code-level difference | Why it changed |
|---|---|---|---|
| Dockerfile | Multi-stage frontend image | Node builds Vite assets; Nginx serves only dist | Smaller, predictable UAT runtime |
| nginx.conf | Static hosting and SPA fallback | Adds /health, immutable asset caching and fallback to index.html | Direct form URLs survive refresh and containers can be health-checked |
| docker-compose-dev.yml | Frontend UAT service | Requires API/WebSocket build arguments and exposes the configured frontend port | Repeatable server deployment |
| .dockerignore | Smaller and safer build context | Excludes node_modules, dist, Git, local env files and legacy code | Faster builds and fewer accidental files in images |
| .env.example | Environment template | Documents GraphQL, WebSocket and port values without real secrets | Shows developers exactly what must be configured |
| .gitignore | Local/generated exclusions | Ignores local credentials, build artifacts and archived reference code as intended | Prevents accidental commits |

### Frontend tests

| Files | What they verify | Why they matter |
|---|---|---|
| src/hooks/use-web-socket.test.tsx | Request IDs, stale responses and lifecycle cleanup | Verifies AI responses cannot be mixed easily |
| src/hooks/use-voice-recorder.test.tsx | Voice/form coordination and manual-edit protection | Protects clinician input |
| src/hooks/use-form-management.test.tsx | Existing report load, reset, Undo and stale initialization | Covers the reset and prefetch bugs reported during testing |
| src/utils/auto-submit-manager.test.tsx | Delay, cancellation, ordering and latest state | Prevents stale automatic saves |
| src/utils/form-submission.test.ts | Common save success/failure behavior | Verifies all buttons use the same contract |
| src/utils/report-payload.test.ts | Zero, null, aliases, invalid fields and form wrapping | Covers the GraphQL validation failure seen in UAT |
| src/utils/first-assessment.test.ts | API load/save round trip | Prevents reload shape corruption |
| src/reducers/form-renderer.reducer.test.ts | Nested field/array updates | Protects the generic renderer |
| test/SimpleFormTests.test.tsx | Active SNC/Physio renderer operations | Replaces tests aimed at removed components with active UI coverage |

## Backend changes

### Request processing and runtime

| File | What was implemented | Code-level difference | Why it changed |
|---|---|---|---|
| main.py | Shared text/audio form pipeline | Both request types use process_form_with_input, formKey is forwarded, input sizes are checked, rate admission occurs before processing and request IDs are echoed | Keeps all entry paths consistent and prevents the wrong form processor |
| src/runtime.py | Bounded execution and follow-ups | Adds a limited worker pool, queue/call timeouts, RequestSocket and cancellable suggestion/recommendation tasks | Prevents blocking calls from freezing the event loop or growing without limit |
| src/forms/registry.py | Central form routing | Registers Assessment and First Assessment strategies, sections and defaults; unknown forms use legacy behavior | Makes form behavior explicit and easier to extend |
| src/llm/functionalities.py | Unified/grouped form processing | Dispatches by registry, processes First Assessment in two parallel groups, validates merged output and rejects malformed/partial group results | Reduces AI calls while improving correctness |

### AI, prompts and cost control

| File | What was implemented | Code-level difference | Why it changed |
|---|---|---|---|
| src/model_clients.py | Immutable model client cache | Creates clients by model and output limit instead of mutating a shared client | Prevents concurrent requests from changing each other's model settings |
| src/llm/llm_manager.py | Routed models and safer fallback | Centralizes primary/fast/fallback calls, usage extraction and truncation checks | Improves provider reliability and makes token usage visible |
| src/prompts.py | Clinical grounding rules | Adds current-note priority, exercise/load mapping, side-measurement and goal rules | Reduces incorrect field placement and hallucinated goals |
| src/suggestion_validation.py | Evidence-based suggestions | Requires path, evidence and question; filters populated or unsupported targets | Stops generic empty-field checklists and unsupported suggestions |
| src/forms/measurement_cleanup.py | Side-value deduplication | Removes a duplicate general value only for newly changed single-sided measurements | Keeps right/left measurements accurate without modifying unrelated rows |
| src/forms/goal_cleanup.py | Goal-intent guard | Preserves existing goals unless the current note explicitly states goal intent | Stops current measurements from becoming invented goals |

### Validation, Redis and persistence support

| File | What was implemented | Code-level difference | Why it changed |
|---|---|---|---|
| src/schemas/validation.py | Typed clinical validation | Distinguishes blank, null, zero and decimal values; rejects booleans, non-finite numbers and invalid RPE; preserves allowed extra metadata | Prevents silent numeric corruption |
| src/schemas/load_values.py | Conservative load normalization | Converts numeric strings and simple number words while retaining named resistance | Makes values such as twenty kilograms serialize consistently |
| src/redis_handler.py | Efficient history reads | Fetches only the required recent entries | Reduces unnecessary Redis data transfer |
| src/redis_services.py | Atomic rate admission | Replaces separate get/check/increment operations with one Redis Lua operation | Prevents concurrent requests bypassing the limit |
| src/mongo_store.py | Optional Mongo behavior | Skips unconfigured Mongo cleanly and avoids logging connection details | Prevents noisy localhost connection attempts and credential exposure |

### Backend deployment files

| File | What was implemented | Code-level difference | Why it changed |
|---|---|---|---|
| Dockerfile | Safer server logging | Runs Uvicorn at INFO instead of WebSocket payload-level DEBUG | Reduces sensitive request logging |
| docker-compose.yml | Restricted production defaults | Requires explicit frontend origin, disables full/debug logging and disables debug endpoints | Makes deployment safer for clinical data |
| .env.example | Optimization/runtime settings | Documents workers, timeouts, limits, model routes and optional features | Gives deployment teams one configuration reference |
| .gitignore | Legacy and generated exclusions | Tracks manifests/readmes while excluding runtime data and archived implementation files from active tooling | Keeps repository behavior predictable |

### Backend tests

| Files | What they verify | Why they matter |
|---|---|---|
| tests/test_optimization.py | Shared handlers, worker limits, request IDs, model routing and fallbacks | Covers the main reliability changes |
| tests/test_form_correctness.py | Pydantic validation, grouped failure and routing | Prevents partial invalid forms |
| tests/test_measurement_cleanup.py | Side/general value rules including zero and bilateral cases | Covers the duplicated measurement bug |
| tests/test_goal_cleanup.py | Explicit goal intent and preservation | Covers invented-goal regression |
| tests/test_suggestions.py | Evidence and empty-field filtering | Verifies suggestion quality controls |
| tests/test_redis_admission.py | Atomic concurrent rate limiting | Checks race-free request admission |

## Removed active files and legacy preservation

These files were removed from active source because they were duplicate,
unrouted or historical implementations. They are preserved under each
repository's legacy directory with original paths and SHA-256 hashes.

### Frontend

| Removed path | Active replacement/reason |
|---|---|
| src/pages/assesment-page.tsx | Unrouted, misspelled historical page; src/pages/form-page.tsx is active |
| src/components/forms/assessment-form.tsx | Replaced by schema-driven form-renderer.tsx |
| src/components/forms/assessment-form-section.tsx | Replaced by generic field/section rendering |
| src/components/forms/form-section.tsx | Active wrapper moved to src/components/sections/form-section.tsx |
| src/components/forms/physio-form-renderer.tsx | Generic renderer now covers the active form structure |
| src/components/forms/section-transcription-box.tsx | Current audio/transcription components handle this flow |
| .env.local and .env.production | Removed from Git because credentials/config belong on each machine/server |

### Backend

| Removed path group | Active replacement/reason |
|---|---|
| functionalities.py and src/llm/old_functionalities.py variants | Active implementation is src/llm/functionalities.py |
| src/old_prompts.py variants | Active prompt source is src/prompts.py |
| src/llm/phoenix/vertex.py | Model tracing/client logic is handled by the active manager/client flow |

The archives are reference material, not a second runnable application.

## Short client-review explanation

> The PR consolidates duplicate frontend and backend paths into one active
> schema-driven flow. It makes voice/text requests traceable, protects manual
> edits, validates AI output, reduces First Assessment model calls, unifies
> GraphQL saves and adds repeatable Docker deployment. Historical implementations
> remain available in legacy folders, and regression tests cover the reported
> reset, measurement, goal, socket and GraphQL payload issues.

## Important review limits

- The browser-visible WebSocket/API keys are configuration, not user identity.
  Gateway/session authorization is still required.
- Cross-client save conflicts require server-side version enforcement.
- AI-generated clinical content still requires clinician review.
- SNC and Physio retain legacy-compatible backend processing and do not yet have
  the same validation depth as the registered Assessment flows.
