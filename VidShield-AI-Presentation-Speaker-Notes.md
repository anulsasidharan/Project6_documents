# VidShield AI — Slide deck outline & speaker notes

Use this alongside **`VidShield-AI-Presentation-Study-Guide.md`**. Each block is one slide: **on-screen bullets** (minimal) and **what you say** (fuller).

**Timing:** ~12 minutes core + 3 minutes demo + Q&A. Adjust slide count for your slot.

---

## Slide 1 — Title

**On screen:** VidShield AI — AI Video Intelligence & Content Moderation Platform — [Your name] — [Date]

**Speaker notes:** Good morning. I am presenting VidShield AI, a platform we designed for organizations that need to understand and moderate video at scale—recorded libraries and live streams—with policy-aware AI, operator workflows, and full auditability.

---

## Slide 2 — The problem

**On screen:** Manual review does not scale · Policy varies by market · Incidents are expensive · Partners need structured signals

**Speaker notes:** Trust-and-safety teams face a structural problem: video volume grows faster than reviewer headcount. Policies differ by product and jurisdiction. When something goes wrong, the cost is reputational and regulatory. Downstream systems do not want raw video; they want decisions, timestamps, and webhook events they can act on.

---

## Slide 3 — What VidShield is

**On screen:** Web platform + API · Upload & presigned S3 · AI pipeline · Moderation queue · Live streams & alerts · Analytics · Webhooks · Billing & audits

**Speaker notes:** VidShield is not a single model call. It is a product: a Next.js operator experience, a FastAPI backend under `/api/v1`, asynchronous workers for heavy work, PostgreSQL as the system of record, and S3 for media. Operators get queues and dashboards; the system can notify partners via webhooks and support compliance with audit logs.

---

## Slide 4 — Who uses it

**On screen:** Operator / API consumer — day-to-day moderation & uploads · Admin — users, audits, support, billing metrics

**Speaker notes:** We use role-based access. Operators and API consumers manage videos, policies, and the moderation queue. Administrators additionally manage users, read access and moderation audits, inspect agent audit trails, and handle support and billing visibility.

---

## Slide 5 — High-level architecture

**On screen:** Browser → CloudFront / ALB → Next.js + FastAPI · Celery workers · PostgreSQL · Redis · S3 · OpenAI + Pinecone

**Speaker notes:** Requests hit our edge and load balancer. The browser talks to Next.js; in production the app often calls the same origin and Next rewrites to the API. Anything expensive—frame extraction, transcription, LangGraph runs—runs in Celery workers so HTTP stays responsive. Redis backs the broker and rate limits. Blobs live in S3 with presigned access. AI calls go to OpenAI; vector similarity uses Pinecone when configured.

---

## Slide 6 — Request path (one slide)

**On screen:** Router → Service → DB · Middleware: CORS, rate limit, context · Success: `{ "data": ... }` · Errors: structured envelope

**Speaker notes:** We follow clean architecture: routes stay thin, services hold domain logic, and the database is accessed through our ORM layer. Middleware applies cross-cutting concerns. Clients always get a consistent JSON shape, which matters for partners and for debugging.

---

## Slide 7 — Why async (Celery)

**On screen:** `process_video` → extract frames → transcribe → thumbnail → LangGraph analysis · Dedicated queues: video, moderation, analytics, …

**Speaker notes:** Video understanding is minutes of CPU and IO, not milliseconds. We enqueue a pipeline task after upload. Workers run frame extraction—FFmpeg and OpenCV paths—then Whisper-style transcription, then the AI graph. Separate queues let us scale and isolate failure domains.

---

## Slide 8 — The AI spine (memorize this)

**On screen:** **O → (C + S + M) → S → R** — One opening · Three parallel lenses · One verdict · One report

**Speaker notes:** The heart of recorded-video intelligence is a LangGraph workflow. The Orchestrator prepares frames and transcript. Three specialists run in parallel: Content analysis with vision, Scene classification, and Metadata extraction including OCR-style signals. The Safety Checker fan-ins all three and applies active policy rules. The Report Generator produces the structured moderation artifact we store and show in the UI.

---

## Slide 9 — Agent roster (table slide)

**On screen:** Short table: Orchestrator — prep · Content — vision + summary · Scene — categories · Metadata — entities/OCR · Safety — policy + violations · Report — synthesis · Live moderator — fast chunk path

**Speaker notes:** Each agent has a single responsibility. The orchestrator does not judge policy; the safety checker does. The report generator does not re-run vision; it synthesizes. For live streams we use a dedicated live moderator that caps frames for latency and can run OCR and face-related checks in parallel for alerts.

---

## Slide 10 — Second graph: fast moderation workflow

**On screen:** load context → moderation chain → confidence check · Low confidence → escalate to human · High confidence → finalize

**Speaker notes:** Not every decision needs the full graph. We have a lighter LangGraph workflow for re-moderation, policy updates, or segments where context already exists. If model confidence is below a threshold—default point six—we escalate rather than silently guessing. That is how we balance automation with operational risk.

---

## Slide 11 — Tools vs agents

**On screen:** Agents reason · Tools perceive — frames, Whisper, OCR, object detection, Pinecone similarity, face analysis (live)

**Speaker notes:** We do not stream entire videos into prompts. We sample frames and transcribe audio. Tools encapsulate FFmpeg, OpenAI, and optional Pinecone retrieval. This keeps costs predictable and boundaries testable.

---

## Slide 12 — End-to-end: recorded video

**On screen:** Upload to S3 → API creates row → Celery pipeline → graph → DB → queue & webhooks

**Speaker notes:** Walk the happy path slowly once: the user obtains a presigned URL and uploads. The API records metadata and enqueues work. Workers extract frames and text, run the graph with the tenant’s active policies, persist results, and the moderation queue updates. Webhooks can fire for downstream automation.

---

## Slide 13 — Live stream use case

**On screen:** Ingest frames → Live Stream Moderator → violations & severity → alerts → WebSocket to dashboard

**Speaker notes:** For live content, latency matters. We ingest frame batches, score them quickly with strict caps, write alerts, and push updates over WebSockets so operators see problems as they unfold. Same trust principles as VOD, different performance envelope.

---

## Slide 14 — Trust, security, compliance hooks

**On screen:** JWT + RBAC · Rate limits · Presigned S3 · Stripe webhook verification · Access / moderation / agent audits · Structured logging

**Speaker notes:** Security is layered: authentication and roles, rate limiting, least-privilege storage access, and verified billing webhooks. For governance we persist access audits, moderation audits, and agent audit entries so leadership can answer what the system decided and why.

---

## Slide 15 — Business value (no fake dollars)

**On screen:** Reviewer productivity · Time to detect · Decision consistency · Compliance evidence · Partner integration speed

**Speaker notes:** I will not quote a dollar ROI without your organization’s data. The value categories are standard: more items cleared per reviewer hour, faster detection on live streams, fewer contradictory decisions thanks to policy grounding, stronger audit evidence, and faster partner onboarding via APIs and webhooks. In pilot you measure auto-triage rate, escalation rate, and mean time to first decision.

---

## Slide 16 — Tech stack (optional dense slide)

**On screen:** Next.js 14 · FastAPI · PostgreSQL · Redis · Celery · S3 · LangGraph · OpenAI · Pinecone · Terraform / ECS / GitHub Actions

**Speaker notes:** This is a modern full-stack plus worker topology, deployed on AWS in our reference design—ECS Fargate, RDS, ElastiCache, S3, CloudFront—with infrastructure as code and CI/CD from GitHub Actions.

---

## Slide 17 — Demo or screenshot

**On screen:** [Screenshot: dashboard / video detail / moderation queue / live alerts]

**Speaker notes:** Here is the product in action: upload or select a video, watch status transition, open the moderation outcome, and optionally show a live alert. If live demo is risky, screenshots are fine.

---

## Slide 18 — Honest scope (builds credibility)

**On screen:** API keys: managed in UI; protected routes use JWT today · Multi-tenant: partial—verify per route · Mobile: out of repo

**Speaker notes:** I want to be precise. API keys exist for developer workflows but the dependency layer we inspected uses Bearer JWT on protected routes—so position keys as managed credentials with a clear integration roadmap. Multi-tenant fields exist; enforcement depth varies by endpoint—something we document and harden over time. There is no mobile app in this repository.

---

## Slide 19 — Closing

**On screen:** VidShield turns unstructured video into policy-grounded decisions—with queues, audits, webhooks, and ops visibility.

**Speaker notes:** Thank you. The thesis is simple: scale video trust and safety the same way we scale other software—with async pipelines, explicit orchestration, and operator-grade product surfaces. I am happy to take questions.

---

## Slide 20 — Backup: Q&A cues

**On screen:** Why LangGraph? · Why Celery? · Model cost? · Human in the loop?

**Speaker notes:** LangGraph gives us an explicit state machine for multi-agent flows. Celery decouples heavy work from HTTP and gives retries. We cap frames and tune models to control cost. Humans enter when confidence is low or policy requires review—we encode that in the moderation workflow graph.

---

## Delivery checklist (print or keep on phone)

- [ ] Memorize **O → (C+S+M) → S → R** and say the rhyme once.  
- [ ] One architecture diagram from `docs/AWS-ARCHITECTURE-DESIGN.md` ready.  
- [ ] Demo path rehearsed: login → video → status → moderation.  
- [ ] Backup slide for API keys / JWT honesty.  
- [ ] Water, slow down on fan-out/fan-in, pause after thesis sentence.
