# Clinician Agent — Number-wise Implementation Explanation (Hinglish)

Is document ka purpose hai ki developer ya client jab PR review kare, tab hum
har implementation ko simple language mein explain kar saken. Har point mein
problem, files, code change, runtime effect aur short explanation diya gaya hai.

## 1. Legacy Code Preservation

**Problem:** Purana code active code ke saath mixed tha. Isse duplicate
implementations aur correct file identify karne mein confusion hota tha.

**Main files/folders:**

- Clinician-agent/legacy/
- clinician-agent-FE-v2/legacy/
- Dono repositories ke legacy/manifest.json
- Dono repositories ke legacy/README.md

**Code change:** Old implementations ko active src paths se hata kar same
structure ke saath legacy folders mein preserve kiya. Manifest mein original
path, archived path aur SHA-256 hash store kiya.

**Runtime effect:** Legacy files build, lint, tests aur runtime imports ka part
nahi hain. Active application sirf current src files use karti hai.

**Client ko simple explanation:** “Purana code delete nahi kiya. Reference aur
rollback understanding ke liye legacy folder mein hash ke saath preserve kiya,
lekin duplicate code ko active runtime se remove kiya.”

## 2. Registry-based Form Routing

**Problem:** Backend ko form type milne ke baad bhi kuch requests default
Assessment processor mein ja sakti thi.

**Main files/functions:**

- Backend src/forms/registry.py — get_form_spec
- Backend main.py — process_form_with_input
- Backend src/llm/functionalities.py — main_processor

**Code change:** formKey ko text aur audio dono request paths se AI processor tak
forward kiya. Registry har supported form ke sections, defaults aur processing
strategy define karti hai.

~~~text
formKey -> get_form_spec(formKey) -> correct processing strategy
~~~

**Runtime effect:** firstAssessment request First Assessment flow mein aur
assessment request follow-up flow mein jaati hai.

**Client explanation:** “Form name hardcode nahi hai. Registry decide karti hai
ki kis form ko kaunse sections aur processor ke through run karna hai.”

## 3. First Assessment Optimization

**Problem:** First Assessment ke har section ke liye separate model call karne
se cost aur latency badh rahi thi.

**Main files/functions:**

- Backend src/forms/registry.py — FIRST_ASSESSMENT_SPEC
- Backend src/llm/functionalities.py — _process_form_mode_grouped
- Backend src/llm/functionalities.py — _fill_form_subset

**Code change:** Seven sections ko two logical groups mein combine kiya:

~~~text
Group 1: clinicalDetails + subjectiveAssessments + objectiveAssessment
Group 2: subjectiveGoals + objectiveGoals + recommendation + patientAdvice
~~~

ThreadPoolExecutor se dono model calls parallel run hoti hain. Dono successful
hone ke baad result merge aur validate hota hai.

**Runtime effect:** Seven calls ki jagah two calls aur approximately one parallel
call window ki latency.

**Client explanation:** “Related fields ko two groups mein process karke AI call
count aur wait time reduce kiya, without mixing unrelated form types.”

## 4. Backend Load Control

**Problem:** Speech aur AI SDK calls synchronous hain. Direct event-loop
execution server ko block kar sakti thi, aur unlimited work memory pressure
create kar sakta tha.

**Main files/functions:**

- Backend src/runtime.py — BoundedWorker
- Backend src/runtime.py — RequestSocket
- Backend main.py — websocket_endpoint
- Backend main.py — process_form_with_input
- Backend src/redis_services.py — rate limiter

**Code change:**

- Fixed-size worker pool add kiya.
- Queue admission timeout aur provider-call timeout add kiya.
- Audio size/duration limits add kiye.
- Paid processing se pehle rate admission check kiya.
- Har browser request ko requestId diya aur response mein return kiya.

**Runtime effect:** Server overload ke time controlled error deta hai. Old aur
new WebSocket responses identify ki ja sakti hain.

**Client explanation:** “Humne concurrency ko bounded rakha hai, isliye sudden
traffic unlimited AI jobs create nahi karta aur har response apni request se
match hota hai.”

## 5. AI Model Client and Fallback

**Problem:** Shared mutable model client concurrent requests ke beech model ya
output settings mix kar sakta tha. Provider failure handling bhi inconsistent
thi.

**Main files/functions:**

- Backend src/model_clients.py
- Backend src/llm/llm_manager.py — complete_routed
- Backend optimization.env.example

**Code change:** Model name aur output limit ke basis par immutable clients
create/cache kiye. Primary, fast aur fallback routing centralize ki. Empty or
truncated responses aur usage metadata check kiya.

**Runtime effect:** Ek request ka model configuration doosri request ko affect
nahi karta. Configured fallback controlled way mein use hota hai.

**Client explanation:** “Model selection central aur request-safe hai. Primary
provider issue par configured fallback use hota hai, aur token usage observable
hai.”

## 6. Data Validation

**Problem:** Blank, null aur zero ko same treat karne se valid zero measurement
lose ho sakta tha. Invalid RPE/non-numeric values silently pass ho sakte the.

**Main files/functions:**

- Backend src/schemas/validation.py — validate_complete_form
- Backend src/schemas/load_values.py — normalize_load
- Frontend src/utils/report-payload.ts — measurement

**Code change:**

- Blank, null, zero aur decimal ko separately handle kiya.
- Boolean, NaN, Infinity aur invalid measurements reject kiye.
- RPE ko 0–10 range mein validate kiya.
- Number words such as twenty ko safe numeric string mein normalize kiya.
- Named resistance such as red band ko unchanged rakha.

**Runtime effect:** Valid zero/decimal save hote hain aur invalid clinical
numbers API tak nahi jaate.

**Client explanation:** “Validation data ko silently guess nahi karti. Zero ko
valid value rakhti hai aur invalid numeric/RPE input ko reject karti hai.”

## 7. Measurement Duplication Fix

**Problem:** Right knee 130 jaise measurement kabhi general value aur right value
dono mein duplicate ho raha tha.

**Main files/functions:**

- Backend src/forms/measurement_cleanup.py — normalize_changed_measurements
- Backend src/llm/functionalities.py
- Backend tests/test_measurement_cleanup.py

**Code change:** Sirf newly changed, single-sided row inspect hota hai. Agar
general value aur populated side same hain aur separate overall measurement
dictate nahi hui, to redundant general value clear hoti hai.

**Safety:** Existing unchanged rows, bilateral values, conflicting labels aur
explicit overall/general measurements preserve hote hain.

**Client explanation:** “130 right knee mein store hota hai; same 130 general
field mein duplicate nahi hota. Existing unrelated measurements touch nahi
hote.”

## 8. Incorrect Goal Prevention

**Problem:** “Correct knee flexion to 130” ko AI current measurement ke saath
objective goal bhi bana raha tha.

**Main files/functions:**

- Backend src/forms/goal_cleanup.py — preserve_unrequested_goals
- Backend src/prompts.py — goal grounding rules
- Backend src/llm/functionalities.py
- Backend tests/test_goal_cleanup.py

**Code change:** Current transcript mein explicit goal intent na ho to existing
subjectiveGoals aur objectiveGoals preserve hote hain; model-created new goals
accept nahi hote.

**Runtime effect:** Measurement correction measurement section mein rehti hai.
Goal tabhi update hota hai jab clinician goal/target/aim jaisi explicit language
use kare.

**Client explanation:** “Current finding aur future goal ko separate rakha hai.
AI sirf explicit goal instruction par goal create/update karta hai.”

## 9. Clarification Suggestions

**Problem:** AI populated fields ke liye bhi generic questions suggest kar sakta
tha aur transcript evidence clear nahi tha.

**Main files/functions:**

- Backend src/suggestion_validation.py — filter_suggestions
- Backend src/prompts.py — suggestion prompt
- Backend src/llm/functionalities.py — suggestions mode
- Backend tests/test_suggestions.py

**Code change:** Suggestion output ko field path, exact evidence aur question ke
structured format mein manga. Only real empty leaf fields and transcript-backed
evidence accept hota hai. Duplicate suggestions remove hote hain aur count limit
hoti hai.

**Runtime effect:** UI mein focused clarification questions aati hain, empty-field
checklist nahi.

**Client explanation:** “Suggestion ko accept karne se pehle hum check karte hain
ki target field empty hai aur question current note ke evidence se related hai.”

## 10. Manual Edit Protection

**Problem:** Clinician AI processing ke time form edit kare, aur late AI response
us latest edit ko overwrite kar de.

**Main files/functions:**

- Frontend src/hooks/use-voice-recorder.tsx — requestFormSnapshot
- Frontend src/components/forms/form-renderer.tsx — getFormData ref
- Frontend src/hooks/use-web-socket.ts — requestId matching

**Code change:** Request send hote time current form ka snapshot save hota hai.
Response aane par snapshot ko current renderer state se compare kiya jaata hai.
Mismatch par old AI result apply nahi hota.

**Runtime effect:** Clinician ki latest manual edit safe rehti hai.

**Client explanation:** “Late AI response ko blindly merge nahi karte. Form
change hua ho to user edit preserve hoti hai aur instruction dobara process
karne ko bola jaata hai.”

## 11. Complete Form Reset

**Problem:** Reset ke baad visible values ya prefetched values wapas aa rahi thi,
kyunki parent data, renderer state aur voice state ek saath clear nahi ho rahe
the.

**Main files/functions:**

- Frontend src/hooks/use-form-management.tsx — handleFormReset
- Frontend src/pages/form-page.tsx — resetVersion key
- Frontend src/hooks/use-voice-recorder.tsx — resetSession
- Frontend src/utils/schema-utils.ts — defaultStateFromSchema

**Code change:**

- Fresh blank state schema se create hota hai.
- resetVersion renderer ko clean remount/reset deta hai.
- Pending auto-save cancel hota hai.
- Old socket request, transcript and suggestions clear hote hain.
- Late prefetch response epoch check se ignore hota hai.
- Previous draft ke liye guarded Undo diya.

**Runtime effect:** Reset all active UI state clear karta hai. Database mein
clear persist karne ke liye clinician ko Save karna hota hai.

**Client explanation:** “Reset sirf input text clear nahi karta; renderer,
prefetch guard aur voice session ko bhi reset karta hai.”

## 12. Common Save Flow

**Problem:** Bottom Save, renderer save aur auto-save different payload logic use
kar rahe the.

**Main files/functions:**

- Frontend src/utils/form-submission.ts — submitFormData
- Frontend src/handlers/form-submit-handlers.ts
- Frontend src/hooks/use-form-management.tsx
- Frontend src/utils/graphql-client.ts — saveReport

**Code change:** Manual aur automatic save paths ko submitFormData, saveReport
aur buildReportPayload chain ke through route kiya.

~~~text
UI save trigger -> submitFormData -> saveReport -> buildReportPayload -> GraphQL
~~~

**Runtime effect:** Same form ke different save buttons same API shape use karte
hain.

**Client explanation:** “Save logic ek jagah centralized hai, isliye manual aur
auto-save ke results alag nahi hote.”

## 13. Ordered Save Requests

**Problem:** Same appointment ke two saves parallel finish hokar newer data ko
older data se overwrite kar sakte the.

**Main files/functions:**

- Frontend src/utils/report-payload.ts — queueReportSave
- Frontend src/utils/graphql-client.ts — saveReport
- Frontend src/utils/auto-submit-manager.ts

**Code change:** Appointment ID ke basis par Promise queue maintain ki. Next save
previous save complete/fail hone ke baad start hota hai. Failure next save ko
permanently block nahi karti.

**Runtime effect:** Same browser session mein saves correct order mein jaate hain.

**Limit:** Different browser/tab ke conflicts ke liye server-side version check
abhi bhi required hai.

**Client explanation:** “Ek appointment ki browser-side saves serialize hoti
hain, so local race reduce hota hai.”

## 14. Save Confirmation

**Problem:** HTTP request resolve hone ko successful database save assume kiya
ja raha tha.

**Main files/functions:**

- Frontend src/utils/graphql-client.ts — saveReport
- Frontend src/utils/form-submission.ts

**Code change:** GraphQL updateAgentReport response mein report ID verify kiya.
Confirmed ID ke baad hi success toast show hota hai.

**Runtime effect:** Empty/unconfirmed GraphQL response par false “Saved”
notification nahi dikhti.

**Client explanation:** “Success UI tabhi aati hai jab API saved report ID return
karti hai.”

## 15. GraphQL Field Mapping

**Problem:** AI aliases directly GraphQL input mein ja rahe the. API ne name,
details, description aur row-level rpe fields reject kiye.

**Main files/functions:**

- Frontend src/utils/report-payload.ts — buildReportPayload
- Frontend src/utils/report-payload.test.ts
- Frontend src/utils/first-assessment.ts

**Code change:** Strict allow-list serializer add kiya:

~~~text
name        -> testName
details     -> comments
description -> conclusion
~~~

Unknown fields remove hote hain. Frontend objectiveAssessment object API ke
objectiveAssessments array mein convert hota hai.

**Runtime effect:** Dynamic AI output valid GraphQL input type mein convert hota
hai aur unsupported keys API validation fail nahi karte.

**Client explanation:** “AI aur GraphQL ke naming difference ko boundary
serializer handle karta hai; raw AI object database API ko directly nahi bhejte.”

## 16. Existing Report Reload

**Problem:** Existing First Assessment refresh par blank report initialize ho
sakta tha ya API list/object shape renderer se mismatch karti thi.

**Main files/functions:**

- Frontend src/utils/api.ts — fetchFirstAssessmentReport
- Frontend src/utils/first-assessment.ts — firstAssessmentToForm
- Frontend src/hooks/use-form-management.tsx

**Code change:** Patient reports query karke exact appointment match kiya. Existing
agentReport mile to create mutation skip karke saved form restore kiya.
objectiveAssessments list ko renderer objectiveAssessment object mein map kiya.

**Runtime effect:** Saved values refresh/navigation ke baad correctly load hoti
hain.

**Client explanation:** “Pehle existing report lookup hota hai; report milne par
new blank report create nahi hota.”

## 17. Clinical Record Prefill

**Problem:** New working agent report empty ho sakta hai, jabki appointment ke
Report.records mein existing clinical information available hoti hai.

**Main files/functions:**

- Frontend src/utils/api.ts — fetchReportByAppointment
- Frontend src/utils/records-to-form.ts — mapRecordsToForm
- Frontend src/hooks/use-form-management.tsx

**Code change:** Exact appointment ke records query kiye aur different database
names/shapes ko First Assessment or Assessment renderer fields mein map kiya.
Date, test, plan, goals and recommendations conversion helpers add kiye.

**Runtime effect:** Existing agent report authoritative rehta hai. New working
form existing visit records se prefill ho sakta hai.

**Client explanation:** “Saved agent form ho to wahi load hota hai; warna
available clinical record ko form-friendly structure mein map karte hain.”

## 18. Hardcoded Key Removal

**Problem:** API aur WebSocket key fallbacks source code aur Git history mein
present the.

**Main files:**

- Frontend src/utils/api-config.ts
- Frontend src/hooks/use-web-socket.ts
- Frontend vite.config.ts
- Frontend proxy-server.js
- Frontend .env.example

**Code change:** Hardcoded active values ko remove karke VITE_API_KEY aur
VITE_WS_API_KEY environment variables use kiye.

**Runtime effect:** Har environment apni build-time configuration provide karta
hai.

**Important:** VITE values browser bundle mein visible hoti hain. Real user
authorization gateway/session level par required hai. Historical exposed keys
rotate hone chahiye.

**Client explanation:** “Secrets source mein hardcode nahi hain; deployment
environment se aate hain. Browser keys identity security ka replacement nahi.”

## 19. Frontend Docker Deployment

**Problem:** Frontend ke liye repeatable server container configuration nahi thi.

**Main files:**

- Frontend Dockerfile
- Frontend docker-compose-dev.yml
- Frontend .dockerignore
- Frontend UAT_DOCKER_DEPLOYMENT.md

**Code change:** Multi-stage build add kiya. Node stage npm ci aur Vite build
karta hai. Final Nginx image sirf compiled dist serve karti hai. Required VITE
values build arguments ke through validate hote hain.

**Runtime effect:** Same tested image UAT server par consistently run hoti hai.
Node development server production runtime ka part nahi hai.

**Client explanation:** “Build aur runtime separate hain; final container small
static Nginx image hai.”

## 20. SPA Routing

**Problem:** Direct URL ya refresh, for example /firstAssessment/patient/visit,
Nginx par 404 de sakta tha.

**Main file:**

- Frontend nginx.conf

**Code change:** Nginx location fallback add ki:

~~~text
requested static file exists -> serve file
otherwise -> serve index.html -> React Router handles route
~~~

Hashed assets ke liye long cache aur index page ke liye no-cache behavior add
kiya.

**Runtime effect:** Deep links and page refresh work karte hain.

**Client explanation:** “Server unknown application routes ko index.html deta
hai; final page React Router decide karta hai.”

## 21. Health Checks and Safer Deployment Defaults

**Problem:** Container running dikh sakta tha even when application unhealthy
ho. Debug logging clinical payload expose kar sakti thi.

**Main files:**

- Frontend nginx.conf — /health
- Frontend Dockerfile — HEALTHCHECK
- Backend main.py — /health
- Backend Dockerfile
- Backend docker-compose.yml

**Code change:**

- Frontend health endpoint and Docker health command add kiya.
- Backend health Redis, agent and context manager status check karta hai.
- Production Uvicorn logging INFO ki.
- Full payload logging/debug endpoint default off kiya.
- Explicit frontend origin configuration require ki.

**Runtime effect:** Docker/orchestrator unhealthy service detect kar sakta hai,
aur production logs mein raw WebSocket payload exposure reduce hoti hai.

**Client explanation:** “Running process aur healthy application ko separate
check karte hain, aur clinical environment mein debug defaults off hain.”

## 22. Regression Test Coverage

**Problem:** Reported bugs future refactor mein silently return ho sakte the.

**Frontend test files:**

- src/hooks/use-web-socket.test.tsx
- src/hooks/use-voice-recorder.test.tsx
- src/hooks/use-form-management.test.tsx
- src/utils/auto-submit-manager.test.tsx
- src/utils/form-submission.test.ts
- src/utils/report-payload.test.ts
- src/utils/first-assessment.test.ts
- src/reducers/form-renderer.reducer.test.ts
- test/SimpleFormTests.test.tsx

**Backend test files:**

- tests/test_optimization.py
- tests/test_form_correctness.py
- tests/test_measurement_cleanup.py
- tests/test_goal_cleanup.py
- tests/test_suggestions.py
- tests/test_redis_admission.py

**Code change:** Reset, stale fetch, socket response order, manual edit
preservation, save ordering, GraphQL aliases, measurements, goals, suggestions,
worker limits and atomic rate admission ke focused regression cases add kiye.

**Runtime effect:** Future changes ke baad same issue aaye to automated checks
PR/deployment se pehle fail karte hain.

**Client explanation:** “Har important reported issue ke saath regression test
add hua hai, so fix sirf prompt-level claim nahi hai.”

## Complete flow in one line

~~~text
Select patient/appointment
-> load existing report or records
-> render schema-based form
-> record/type note
-> WebSocket
-> Speech-to-Text
-> Gemini grouped processing
-> validation and safeguards
-> clinician review
-> strict GraphQL payload
-> ordered confirmed save
-> report database
~~~

## 60-second PR explanation

> “Is PR mein humne duplicate legacy paths ko archive karke one active
> schema-driven frontend and registry-driven backend flow banaya. Voice and text
> requests ab bounded, traceable aur form-aware hain. First Assessment seven
> separate AI calls ki jagah two parallel grouped calls use karta hai. AI output
> numeric, measurement, goal and suggestion safeguards se validate hota hai.
> Frontend late AI responses se manual edits protect karta hai, complete reset
> karta hai, aur manual/auto save ko strict ordered GraphQL serializer se bhejta
> hai. Existing reports reload hote hain, records prefill supported hai, Docker
> deployment repeatable hai, aur main bugs regression tests se covered hain.”

## Honest limitations to mention

- AI clinical output ko clinician review karna zaroori hai.
- Browser-visible API/WebSocket keys real user authentication nahi hain.
- Cross-browser save conflicts ke liye server-side versioning required hai.
- SNC/Physio backend legacy-compatible path par hain; registered Assessment
  flows jaisi validation depth abhi nahi hai.
- Controlled UAT complete karna production approval ke equal nahi hai.
