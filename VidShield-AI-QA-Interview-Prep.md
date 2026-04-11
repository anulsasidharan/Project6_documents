# VidShield AI — Q&A Bank (Presentation & Interview Prep)

**Purpose:** Rehearse talks, panel Q&A, and technical interviews aligned with this repository.  
**Sources:** `docs/PRD.md`, `docs/ARCHITECTURE.md`, `docs/AWS-ARCHITECTURE-DESIGN.md`, `backend/app/ai/*`, `backend/app/workers/*`, `backend/app/api/*`.  

**Diagrams:** PNGs remain in `docs/architecture_images/` (this document **embeds** them by relative URL; files are **not duplicated**). Paths use `%20` for spaces so more viewers render correctly. If an image still fails, open the PNG from that folder or paste the path into your slide deck.

---

## Table of contents

1. [Architecture figure gallery](#1-architecture-figure-gallery)  
2. [Questions tied to diagrams](#2-questions-tied-to-diagrams)  
3. [General project Q&A — simple](#3-general-project-qa--simple)  
4. [General project Q&A — medium](#4-general-project-qa--medium)  
5. [General project Q&A — hard](#5-general-project-qa--hard)  
6. [Python & backend (FastAPI / Celery / data) coding Q&A](#6-python--backend-fastapi--celery--data-coding-qa)  
7. [AI / LangGraph / LLM coding Q&A](#7-ai--langgraph--llm-coding-qa)  
8. [Behavioral & ownership (short)](#8-behavioral--ownership-short)

---

## 1. Architecture figure gallery

### Figure 1 — AWS cloud (system context)

![Figure 1 — AWS Cloud Architecture](architecture_images/1.%20AWS%20Cloud%20Architecture-l.png)

*Typical story:* Internet users, edge (Route 53, CloudFront, WAF), AWS core (ECS, RDS, Redis, S3), external SaaS (OpenAI, Stripe, etc.).

---

### Figure 2 — VPC

![Figure 2 — VPC Architecture](architecture_images/2.%20VPC%20Architecture-l.png)

*Typical story:* Public subnets (ALB, NAT) vs private subnets (ECS tasks, RDS, Redis); defense in depth.

---

### Figure 3 — VidShield ECS

![Figure 3 — VidShield ECS Architecture](architecture_images/3.%20VidShield%20ECS%20Architecture-l.png)

*Typical story:* Fargate services for API, worker, frontend; target groups behind ALB.

---

### Figure 4 — CloudFront distribution

![Figure 4 — CloudFront Distribution Architecture](architecture_images/4.%20CloudFront%20Distribution%20Architecture-l.png)

*Typical story:* CDN, TLS, origins to ALB and/or S3, caching behavior at edge.

---

### Figure 5 — Edge to ECS

![Figure 5 — AWS Edge to ECS Architecture](architecture_images/5.%20AWS%20Edge%20to%20ECS%20Architecture-l.png)

*Typical story:* Request path from browser through edge to ECS tasks.

---

### Figure 6 — Video upload flow

![Figure 6 — Video Upload Flow](architecture_images/6.Video%20Upload%20Flow-l.png)

*Typical story:* Presigned URL, direct browser → S3, callback/metadata to API.

---

### Figure 7 — Video moderation pipeline

![Figure 7 — Video Moderation Pipeline](architecture_images/7.%20Video%20Moderation%20Pipeline-l.png)

*Typical story:* API enqueues work; workers extract frames, transcribe, run AI graph, persist results, notify.

---

### Figure 8 — WebSocket / realtime

![Figure 8 — Browser to AWS WebSocket Architecture](architecture_images/8.Browser%20to%20AWS%20WebSocket%20Architecture-l.png)

*Typical story:* Sticky sessions / ALB WebSocket support; live stream updates to browser.

---

### Figure 9 — CI/CD

![Figure 9 — CI/CD GitHub to AWS](architecture_images/9.CICD%20Pipeline%20-%20GitHub%20to%20AWS-%20GitHub%20to%20AWS-l.png)

*Typical story:* GitHub Actions → ECR build → ECS deploy; quality gates.

---

### Figure 10 — Observability

![Figure 10 — ECS Fargate Observability](architecture_images/10.ECS%20Fargate%20Observability%20Architecture-l.png)

*Typical story:* CloudWatch logs/metrics/alarms, tracing hooks, operational dashboards.

---

### Figure 11 — Multi-AZ

![Figure 11 — Single Region Multi-AZ](architecture_images/11.Single%20Region%20Multi-AZ%20Architecture-l.png)

*Typical story:* RDS Multi-AZ, redundant NAT/AZ failure domains.

---

### Figure 12 — API & worker split

![Figure 12 — ECS API & Worker Architecture](architecture_images/12.ECS%20API%20Worker%20Architecture-l.png.png)

*Typical story:* Separate ECS services or task definitions for API vs Celery workers; shared data plane (RDS, Redis, S3).

---

## 2. Questions tied to diagrams

### 2.1 Figure 1 (AWS cloud)

**Q:** What major AWS building blocks does VidShield use in production-style hosting?  
**A:** VPC networking, ALB, ECS on Fargate, RDS PostgreSQL, ElastiCache Redis, S3, CloudFront, WAF, ACM, Route 53, Secrets Manager (and optional SQS/Lambda per design docs). Non-AWS: OpenAI, Pinecone, Stripe, SendGrid, Twilio.

**Q:** Why is CloudFront in front of the app?  
**A:** TLS termination at the edge, caching for static assets, geographic latency reduction, and a place to attach WAF policies before traffic hits origins.

---

### 2.2 Figure 2 (VPC)

**Q:** Why place RDS and Redis in private subnets?  
**A:** They should not be directly reachable from the internet; only application tasks in private subnets (or bastion patterns) connect, reducing attack surface.

**Q:** What is the role of NAT Gateway(s)?  
**A:** Allow private subnet tasks (ECS, workers) to reach the internet for package pulls, external APIs (OpenAI), and AWS APIs, without public inbound IPs on those tasks.

---

### 2.3 Figures 3 & 12 (ECS / API vs worker)

**Q:** Why split API and worker into different ECS services?  
**A:** Different scaling profiles (HTTP concurrency vs batch CPU/IO), independent deploys, and blast-radius isolation—worker spikes should not starve API tasks.

**Q:** What do Celery workers do that the API must not?  
**A:** Frame extraction, FFmpeg, Whisper calls, LangGraph runs, long report generation, bulk analytics—anything that would block request threads or exceed HTTP timeouts.

---

### 2.4 Figures 4 & 5 (CloudFront / edge to ECS)

**Q:** How does a browser reach the FastAPI backend in production?  
**A:** Often same-origin through Next.js rewrites (`NEXT_PUBLIC_APP_ENV` production pattern): browser calls `/api/v1` on the site origin; Next forwards to `API_UPSTREAM_URL`. Alternatively direct ALB to API for partner integrations.

**Q:** What ALB considerations matter for WebSockets?  
**A:** Idle timeout, sticky sessions if using Socket.IO across instances, and WebSocket-aware health checks; see `docs/ARCHITECTURE.md` realtime section.

---

### 2.5 Figure 6 (upload flow)

**Q:** Why use S3 presigned uploads instead of posting the file through FastAPI?  
**A:** Offloads bandwidth and disk from API servers, reduces latency, and scales large files better; API only mints short-lived URLs and records metadata.

**Q:** After upload, what triggers processing?  
**A:** Services enqueue Celery tasks (e.g. `process_video` in `video_tasks.py`) with `video_id` and `s3_key`; workers pull from the `video` queue.

---

### 2.6 Figure 7 (moderation pipeline)

**Q:** Walk the moderation pipeline at a high level.  
**A:** Extract frames → transcribe audio (Whisper) → optional thumbnail → run `run_video_analysis` (LangGraph: orchestrator → parallel content/scene/metadata → safety → report) → persist `ModerationResult` / queue entries → optional webhooks and notifications.

**Q:** Where do policy rules enter the pipeline?  
**A:** Loaded into graph state as `policy_rules` (list of dicts) for the Safety Checker agent; policies are CRUD-managed via API and stored in PostgreSQL.

---

### 2.7 Figure 8 (WebSocket)

**Q:** How does live moderation reach the UI?  
**A:** Native WebSocket route under `/api/v1/live/ws/streams/{stream_id}` plus Socket.IO mounted on the ASGI app for room join/leave; frontend uses `socket.io-client` and constants for `WS_URL`.

**Q:** Why both Socket.IO and native WebSocket?  
**A:** Historical/utility split: Socket.IO gives simple room semantics for generic realtime; dedicated WebSocket can carry stream-scoped events with lower overhead depending on client.

---

### 2.8 Figure 9 (CI/CD)

**Q:** How would you describe the deploy path?  
**A:** GitHub Actions builds Docker images, pushes to ECR, updates ECS task definitions, rolls out new tasks—immutable deploy pattern described in `docs/AWS-ARCHITECTURE-DESIGN.md`.

---

### 2.9 Figure 10 (observability)

**Q:** What observability exists in code today?  
**A:** `structlog` for structured logs; Celery task signals; optional `SENTRY_DSN` in settings (verify whether Sentry SDK is initialized in your branch). CloudWatch is the AWS-side sink for container logs/metrics.

---

### 2.10 Figure 11 (Multi-AZ)

**Q:** What failure does Multi-AZ RDS primarily protect against?  
**A:** Availability-zone failure: synchronous standby promotion minimizes downtime versus single-AZ.

**Q:** What does Multi-AZ *not* replace?  
**A:** Backups, point-in-time recovery testing, application-level retries, and chaos testing for Redis broker failover behavior.

---

### 2.11 Cross-diagram drills (interview style)

**Q:** Starting from Figure 6, trace data to Figure 7 on one whiteboard.  
**A:** Browser uploads object to S3 using presigned URL → API records video metadata → Celery `process_video` on worker tier (Figures 3/12) → frame extract + transcribe + `run_video_analysis` (Figure 7) → PostgreSQL + optional webhooks/notifications.

**Q:** Where would you add a virus scan in Figures 5–7?  
**A:** Common pattern: S3 event → Lambda (or worker step post-upload) before heavy processing; keep scan out of the synchronous API path.

**Q:** Relate Figure 8 to Figure 7 without duplicating AI work.  
**A:** Figure 7 is batch/offline graph for full videos; Figure 8 pushes **incremental** live results (chunk moderator, alerts). Share policy rules and audit patterns, not necessarily the full graph per chunk.

**Q:** Which diagram best explains “blast radius” to a CTO?  
**A:** Combine Figure 12 (API vs worker) with Figure 11 (AZ failure): show that API availability can remain acceptable while workers retry in another AZ if designed correctly.

---

## 3. General project Q&A — simple

**Q:** What is VidShield AI in one sentence?  
**A:** A web-backed platform for video ingestion, AI-assisted moderation, live monitoring, analytics, policies, webhooks, billing, and audits.

**Q:** What is the frontend stack?  
**A:** Next.js 14 App Router, React 18, Tailwind, Radix UI, TanStack Query, Zustand, Socket.IO client, Axios.

**Q:** What is the backend stack?  
**A:** Python 3.12, FastAPI, SQLAlchemy 2.0, Alembic, Celery, Redis, PostgreSQL.

**Q:** How do users authenticate?  
**A:** JWT access tokens (Bearer) plus refresh flow; passwords hashed with bcrypt.

**Q:** What roles exist?  
**A:** `admin`, `operator`, `api_consumer` (`UserRole` enum).

**Q:** Where is the REST API versioned?  
**A:** Under `/api/v1/` (see `docs/API_SPEC.md`).

**Q:** What stores videos binary data?  
**A:** AWS S3; metadata in PostgreSQL.

**Q:** Name two specialist AI agents besides the orchestrator.  
**A:** Content Analyzer and Scene Classifier (also Metadata Extractor, Safety Checker, Report Generator).

**Q:** What tool transcribes audio?  
**A:** Whisper-based path in `app/ai/tools/audio_transcriber.py` used from workers/orchestrator.

**Q:** What database holds users and videos?  
**A:** PostgreSQL (see `docs/DB_SCHEMA.md`).

**Q:** What is Stripe used for?  
**A:** Billing: checkout, portal, webhooks updating subscription/payment tables (`docs/PRD.md`).

**Q:** Name two notification channels.  
**A:** Email (SendGrid) and WhatsApp (Twilio); plus in-app notifications.

**Q:** What is the live stream WebSocket path pattern?  
**A:** `/api/v1/live/ws/streams/{stream_id}` (per architecture doc).

**Q:** What is LangChain used for here?  
**A:** Chains such as moderation/insight/summary chains composing LLM calls (`app/ai/chains/`).

**Q:** Where do policies live in the data model?  
**A:** `policies` table with JSON `rules` and `default_action` (PRD).

**Q:** What is `yt-dlp` used for?  
**A:** URL-based video ingestion/analysis flows (PRD stack table).

---

## 4. General project Q&A — medium

**Q:** Why LangGraph instead of a single big prompt?  
**A:** Explicit state machine: parallel fan-out (content, scene, metadata), fan-in to safety, auditable nodes, easier testing and retries per step.

**Q:** Explain the fan-out / fan-in shape.  
**A:** `orchestrator` runs first; edges go to `content_analyzer`, `scene_classifier`, and `metadata_extractor` in parallel; all three connect to `safety_checker`, then `report_generator`, then END.

**Q:** What is the moderation workflow graph for?  
**A:** Fast path when context exists: `load_context` → `run_moderation` (chain) → `evaluate_confidence` → escalate vs finalize; low confidence escalates to human.

**Q:** What is the default confidence threshold in that workflow?  
**A:** `0.6` in `moderation_workflow.py` (configurable per call).

**Q:** How does the API keep JSON responses consistent?  
**A:** Middleware wraps successful JSON as `{"data": ...}`; errors use `{"error": {code, message, details}}` per architecture doc.

**Q:** What middleware order is documented?  
**A:** CORS → RateLimit → RequestContext → DataWrapper (outer to inner comment in `main.py`).

**Q:** Are API keys used to authenticate HTTP requests on protected routes?  
**A:** As documented in PRD: keys are manageable via API, but `deps.py` uses JWT Bearer for `get_current_user`—position honestly for interviews.

**Q:** How does rate limiting work?  
**A:** Redis-backed fixed windows in `app/core/rate_limit.py`; fail-open if Redis is unavailable.

**Q:** What queues does Celery use conceptually?  
**A:** Separate queues for video, moderation, analytics, cleanup, reports, notifications, streams (per PRD/worker modules).

**Q:** What is `trace_id` in the video analysis graph?  
**A:** UUID correlating one pipeline run across `agent_audit` records (`run_video_analysis` sets it in initial state).

**Q:** What does the Report Generator consume?  
**A:** Prior agent outputs in state (content analysis, scene classifications, metadata, safety result) to emit a structured `moderation_report`.

**Q:** How does human review interact with automation?  
**A:** Moderation queue endpoints, review submission, admin overrides (PRD §3.4); workflow graph can escalate on low confidence.

**Q:** What outbound integrations notify external systems?  
**A:** Webhooks (register/test routes), optional email/WhatsApp via notification tasks.

**Q:** Why is Redis used beyond Celery?  
**A:** Rate limiting, refresh-token invalidation patterns, and other ephemeral coordination per PRD/architecture.

**Q:** What does `NEXT_PUBLIC_MOCK_API` do?  
**A:** Toggles mock API routes in the frontend vs Next rewrites to the real backend (`docs/PRD.md`).

**Q:** What is the purpose of `DataWrapper` middleware?  
**A:** Uniform success envelope `{"data": ...}` for most JSON responses (exceptions listed in architecture doc).

**Q:** Where are agent prompts stored?  
**A:** `backend/app/ai/prompts/` (e.g. `moderation_prompts.py`, `analysis_prompts.py`).

---

## 5. General project Q&A — hard

**Q:** How would you defend the system against prompt injection from user-defined policy text?  
**A:** Treat policy text as untrusted: strict schema validation, length limits, allowlisted fields, separate system vs user message roles, log and strip unexpected tool calls, and run policy evaluation in a constrained chain with output validation (`ModerationDecision`, `Violation` models).

**Q:** How do you prevent PII leakage in logs for transcripts?  
**A:** Redact or hash sensitive segments; avoid logging raw transcripts at INFO; use sampling and structured fields; align with GDPR retention in PRD security themes.

**Q:** How would you scale workers horizontally without duplicate side effects?  
**A:** Idempotent tasks keyed by `video_id`, transactional outbox or “processing” status transitions, dedupe keys in Redis, SQS visibility timeouts if moving broker, and unique constraints on moderation rows.

**Q:** What happens if two workers process the same video?  
**A:** Design should use DB row versioning or status checks (`VideoStatus`) so only one pipeline wins; discuss race conditions and how you’d add advisory locks or lease fields.

**Q:** Why TypedDict + `Annotated[..., operator.add]` for some state fields?  
**A:** LangGraph merges partial updates from parallel nodes; list fields that accumulate from concurrent writes need reducers to merge safely.

**Q:** How would you add true API-key auth for partners without breaking web JWT?  
**A:** Second security dependency (API key header) resolving to scopes/tenant; route-level or router-level dependency merge; hash keys at rest (already SHA-256 per PRD); rate limit per key.

**Q:** How does multi-tenant isolation work today?  
**A:** PRD notes `tenant_id` on models but “enforcement depth varies”—answer with verification plan: per-query filters, integration tests per route, and middleware binding tenant from token.

**Q:** Explain failure modes if OpenAI is down mid-graph.  
**A:** Partial `errors` in state, agent audit FAILED entries, fallback `NEEDS_REVIEW` reports, Celery retries with backoff, circuit breaker at HTTP client layer.

**Q:** How would you optimize cost for vision calls?  
**A:** Lower `_MAX_FRAMES_FOR_ANALYSIS`, use `gpt-4o-mini` for some agents, cache embeddings, batch OCR, skip redundant re-runs when policy unchanged, and use the fast moderation workflow for incremental checks.

**Q:** Design a feature flag to disable face analysis in live moderation for privacy regions.  
**A:** Config + policy dimension (e.g. `region=EU` disables `face_analyzer` tool), enforce at tool adapter, log skipped step in agent audit, default safe path to vision+OCR only.

**Q:** How would you implement content retention deletion (GDPR) across S3 and Postgres?  
**A:** Tombstone + async cleanup worker: revoke presigned URLs, delete S3 prefixes by `video_id`, cascade DB rows, purge Redis keys, reindex/delete Pinecone vectors if used.

**Q:** Compare Socket.IO rooms vs stream-scoped WebSocket auth.  
**A:** Rooms are flexible fan-out but need auth on join; dedicated WS can attach `stream_id` and validate token scopes per connection—discuss tradeoffs for authorization testing.

**Q:** What metrics prove the pipeline is healthy in production?  
**A:** Celery queue depth/lag, task failure rate, p95 `run_video_analysis` duration, OpenAI 429 rate, DLQ counts if added, API 5xx, DB connection pool saturation.

---

## 6. Python & backend (FastAPI / Celery / data) coding Q&A

**Q:** What does `Depends(get_current_user)` achieve in FastAPI?  
**A:** Injects the authenticated `User` by parsing `Authorization: Bearer`, validating JWT type `access`, loading the user from DB, and binding structlog context vars.

**Q:** How does `require_role` work?  
**A:** Higher-order dependency returning `_check` that raises `ForbiddenError` if `current_user.role` not in allowed roles—used as `AdminUser` / `OperatorUser` type aliases.

**Q:** Why use `AsyncSession` for API paths but sync session in Celery?  
**A:** FastAPI uses async stack; Celery workers historically use sync SQLAlchemy sessions (`sync_session` in `celery_app.py`) for simpler worker threading model—common split; could migrate workers to async with care.

**Q:** What is `@shared_task(bind=True)` for?  
**A:** Gives task instance `self` for retries (`self.retry(exc=...)`) and access to request id / countdown in `video_tasks.py` patterns.

**Q:** How would you safely pass `video_id` between services?  
**A:** As `uuid.UUID` internally where possible; stringify only at Celery/JSON boundaries; validate with Pydantic schemas on API ingress.

**Q:** Write pseudocode for idempotent “start processing” guard.**  
**A:** `UPDATE videos SET status='processing' WHERE id=:id AND status='uploaded' RETURNING id` — if no row returned, skip duplicate work.

**Q:** What is Pydantic v2’s role in this project?  
**A:** Request/response schemas, settings (`pydantic-settings`), and structured AI outputs (e.g. chain output models).

**Q:** How does Alembic fit the release process?  
**A:** Migrations version the schema; CI should run `alembic upgrade head` against a test DB; production uses migrate job or init container per `docs/DEPLOYMENT.md` / k8s patterns.

**Q:** Name a good unit test target in `video_tasks`.  
**A:** Pure helpers (e.g. `_s3_url`) and orchestration with mocked `extract_frames`, `transcribe_audio`, and `run_video_analysis`.

**Q:** How would you mock S3 in tests?  
**A:** `moto` or stub boto3 client; never hit real buckets in unit tests.

**Q:** Show how you’d declare a FastAPI route dependency for admin-only DELETE.  
**A:** `async def delete(..., user: AdminUser):` where `AdminUser` expands to `Depends(require_role(UserRole.ADMIN))`.

**Q:** Why might you use `HTTPBearer(auto_error=False)`?  
**A:** To distinguish missing vs invalid bearer credentials and return consistent `UnauthorizedError` messages from `get_current_user`.

**Q:** What is the risk of `sync_session` in Celery with long transactions?  
**A:** Locks and pool exhaustion—keep transactions short, commit per task unit, avoid holding DB connections across LLM calls (do LLM work then persist).

**Q:** How would you add request correlation IDs across API and workers?  
**A:** Middleware sets `X-Request-ID` in structlog context; pass same id into Celery task kwargs; propagate to `trace_id` or parallel field in agent audit.

**Q:** Write a pytest test idea for `get_current_user` token type check.  
**A:** Mint JWT with `type: refresh` → expect `UnauthorizedError` “Invalid token type.”

**Q:** How does boto3 S3 client get credentials in AWS vs locally?  
**A:** In AWS: IAM task role via instance metadata; locally: env vars or shared credentials file—never commit keys.

---

## 7. AI / LangGraph / LLM coding Q&A

**Q:** What is `VideoAnalysisState`?  
**A:** A `TypedDict` defining LangGraph shared state: inputs (`video_id`, `video_url`, `policy_rules`), frames/transcript, per-agent outputs, and reducer-backed `errors` / `completed_agents` lists.

**Q:** What does `await _compiled_graph.ainvoke(initial_state)` return?  
**A:** Final merged state dict; `run_video_analysis` reads `moderation_report` and maps to `ModerationReport` Pydantic model.

**Q:** Why pass `frames` and `transcript` optionally into `run_video_analysis`?  
**A:** Skips re-extraction when workers already computed artifacts—composition for efficiency.

**Q:** How does `run_audited_pipeline_step` help operations?  
**A:** Wraps each node to persist agent audit entries with summaries and status for admin visibility (`pipeline_agent_audit.py`).

**Q:** What model does `ContentAnalyzerAgent` use (as coded)?  
**A:** `gpt-4o` with a cap on frames sent to vision (`_MAX_FRAMES_FOR_ANALYSIS`).

**Q:** How does the Live Stream Moderator differ from the offline graph?  
**A:** Chunk-level, strict frame caps for latency, parallel visual + OCR + face analysis, outputs violations/severity for alerts—not the full six-node offline graph.

**Q:** What LangGraph pattern is used for conditional routing in moderation workflow?  
**A:** `add_conditional_edges` with a router function returning `"escalate"` or `"finalize"` based on confidence vs threshold.

**Q:** How would you unit-test a LangGraph node in isolation?  
**A:** Import the async node function, pass minimal `state` dict, `pytest.mark.asyncio`, assert partial update keys; mock LLM client at agent level.

**Q:** What is structured output and why use it here?  
**A:** JSON/schema constrained responses parsed into `SafetyResult`, `ModerationReport`, etc.—reduces free-text drift and makes persistence/UI safe.

**Q:** How do you mitigate hallucinated violations?  
**A:** Cross-check with deterministic signals (scene classifier counts, OCR keywords), confidence gating, `NEEDS_REVIEW` defaults, and human escalation path.

**Q:** Implementing a new agent node—what steps?  
**A:** Subclass `BaseAgent`, add prompts in `app/ai/prompts/`, wire node in `StateGraph`, add audited wrapper, extend `VideoAnalysisState`, update worker/service to populate any new fields, add tests.

**Q:** What does `StateGraph.add_node` expect?  
**A:** A string name and a callable async function `(state) -> partial_state_dict` that LangGraph merges into shared state.

**Q:** Why compile the graph once at import (`_compiled_graph = _build_graph()`)?  
**A:** Amortize build cost; reuse immutable compiled graph across requests in worker processes.

**Q:** How would you add retries for a single node without retrying the whole graph?  
**A:** Wrap node runner with try/except and partial state update, or LangGraph retry policies where supported; alternatively move flaky IO to a tool with tenacity and keep node pure.

**Q:** What’s the difference between `moderation_chain` and `moderation_workflow`?  
**A:** Chain is an LCEL-style LLM composition; workflow is a LangGraph state machine adding confidence routing and escalation semantics.

**Q:** How would you stream partial LLM tokens to the UI?  
**A:** Not implemented in core moderation path—would need SSE/WebSocket endpoint, async iterator from OpenAI stream API, and strict auth; not appropriate for worker-only tasks without redesign.

**Q:** Pinecone: when would similarity search run in this codebase?  
**A:** Via `similarity_search` tool when agents need reference matching—discuss metadata filters and score thresholds.

**Q:** How do you version prompts safely?  
**A:** Git-tracked files, optional prompt version field in audit logs, A/B via config flags, and never hot-edit production prompts without deploy.

---

## 8. Behavioral & ownership (short)

**Q:** Tell me about a tradeoff you made in this architecture.  
**A:** Sample answer: “We capped vision frames to control cost and latency, accepting that rare long-tail events might need re-run with denser sampling or human review.”

**Q:** How did you validate the AI pipeline?  
**A:** Golden tests on mocked LLM outputs, audit log inspection, staged rollout, manual review queue metrics.

**Q:** What would you improve next with two weeks?  
**A:** API-key scoped auth, stronger tenant filters, Sentry wiring if missing, integration tests for live WebSocket path, canary deploys.

---

## Appendix A — Quick diagram → topic map (for flashcards)

| # | File (under `docs/architecture_images/`) | Memorize |
|---|-------------------------------------------|----------|
| 1 | `1. AWS Cloud Architecture-l.png` | System context |
| 2 | `2. VPC Architecture-l.png` | Public vs private |
| 3 | `3. VidShield ECS Architecture-l.png` | Fargate services |
| 4 | `4. CloudFront Distribution Architecture-l.png` | CDN + origins |
| 5 | `5. AWS Edge to ECS Architecture-l.png` | Request path |
| 6 | `6.Video Upload Flow-l.png` | Presigned S3 |
| 7 | `7. Video Moderation Pipeline-l.png` | Workers + AI |
| 8 | `8.Browser to AWS WebSocket Architecture-l.png` | Live updates |
| 9 | `9.CICD Pipeline - GitHub to AWS- GitHub to AWS-l.png` | CI/CD |
| 10 | `10.ECS Fargate Observability Architecture-l.png` | Ops |
| 11 | `11.Single Region Multi-AZ Architecture-l.png` | HA |
| 12 | `12.ECS API Worker Architecture-l.png.png` | Split compute |

---

## Appendix B — Regenerate Word/PDF (optional)

The generated Word file is **`docs/VidShield-AI-QA-Interview-Prep.docx`** (create/update with Pandoc). See `docs/BUILD-PRESENTATION-DOCS.md` for the exact command. You can also merge with the study guide:

```bash
pandoc docs/VidShield-AI-Presentation-Study-Guide.md docs/VidShield-AI-QA-Interview-Prep.md -o docs/VidShield-Interview-Bundle.docx --toc -s --resource-path=docs
```

---

*End of document. Add your own “company-specific” answers (SLAs, team size, metrics from your deployment) in a private copy before interviews.*
