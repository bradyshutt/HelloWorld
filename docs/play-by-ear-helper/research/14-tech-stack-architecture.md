# Tech Stack & Architecture

## Summary

For the Play by Ear Helper MVP, we recommend a **Next.js 15 (App Router) PWA** frontend with **OpenSheetMusicDisplay (OSMD)** for notation, talking to a **FastAPI** Python backend that offloads GPU-heavy transcription work to **Modal** serverless GPU functions. Audio uploads are stored in **Cloudflare R2** (zero-egress), job metadata in a managed **PostgreSQL** (Neon or Supabase), and progress is streamed back to the browser over **Server-Sent Events**. Auth is handled by **Clerk**, observability by **Sentry** (with optional OpenTelemetry instrumentation in FastAPI). This stack minimizes infra ops, keeps GPU costs proportional to use, and lets a small team ship a polished web-first experience that works on mobile via PWA install.

## Recommended MVP Stack

- **Frontend**: Next.js 15 (App Router) + TypeScript + Tailwind, deployed on Vercel. PWA-enabled for mobile install. `MediaRecorder` API for microphone capture, `<input type="file">` for uploads. **OpenSheetMusicDisplay (OSMD)** to render MusicXML returned from the backend. (VexFlow underlies OSMD; we don't render notes by hand.)
- **Backend (API)**: **FastAPI** on Python 3.12, deployed as a container on Fly.io or Railway (or Vercel/Modal web endpoints). Handles auth, signed upload URLs, job orchestration, SSE progress, and serving MusicXML.
- **ML Inference**: **Modal** serverless GPU functions (A10G or L4 for MVP, H100 if needed). Models invoked: Demucs (source separation, optional), then a transcription model (Basic Pitch for piano MVP, MT3 / hybrid CNN-Transformer later) that emits MIDI; `music21` converts MIDI to MusicXML.
- **Storage**: **Cloudflare R2** for raw audio uploads and rendered MusicXML/MIDI artifacts (zero egress, S3-compatible API). Presigned PUT URLs from the FastAPI backend.
- **Queue / Orchestration**: **Modal-native** (function calls + `Function.spawn` for async). Avoid Celery for the MVP — Modal handles autoscaling, retries, and dead-lettering. If we need cross-task DAGs later, add Celery + Redis.
- **Database**: **PostgreSQL on Neon** (serverless, branchable). Stores users, jobs, audio metadata, transcription results pointers.
- **Auth**: **Clerk** — fastest Next.js integration, prebuilt UI, generous free tier (10k MAU), and JWT verifiable from the FastAPI backend.
- **Realtime/Streaming**: **Server-Sent Events** from FastAPI — one-way progress (`queued` → `separating` → `transcribing` → `engraving` → `done`) is a textbook SSE use case. Falls back gracefully through CDNs and proxies; no socket gymnastics.
- **Audio preprocessing**: **ffmpeg** (decode/normalize/resample to 16 kHz mono WAV) inside the Modal image. **librosa** for any feature engineering not provided by the model. **torchaudio** is in maintenance mode as of 2.9, so prefer librosa/ffmpeg for non-tensor work.
- **Observability**: **Sentry** for errors + tracing on both Next.js and FastAPI. Add **OpenTelemetry** instrumentation (`opentelemetry-instrumentation-fastapi`) and ship to Sentry's OTLP endpoint or a free Grafana Cloud tier.
- **CI/CD**: GitHub Actions → Vercel (frontend), Modal `modal deploy` (GPU functions), Fly/Railway image deploy (API).

## Architecture Diagram (Text)

```
 ┌────────────────────────────┐
 │  Browser / PWA (Next.js)   │
 │  - MediaRecorder capture   │
 │  - File upload UI          │
 │  - OSMD renders MusicXML   │
 │  - SSE listener for status │
 └──────────────┬─────────────┘
                │ 1. Clerk JWT auth
                │ 2. POST /jobs  (returns presigned R2 URL + jobId)
                │ 3. PUT audio  ──────────────► Cloudflare R2
                │ 4. POST /jobs/{id}/start
                ▼
 ┌────────────────────────────┐
 │  FastAPI on Fly/Railway    │
 │  - Auth (Clerk JWKS)       │
 │  - Job CRUD → Postgres     │
 │  - Modal Function.spawn()  │
 │  - GET /jobs/{id}/events   │ ← SSE stream
 └──────────────┬─────────────┘
                │ Function.spawn(jobId, r2_key)
                ▼
 ┌────────────────────────────────────────────┐
 │ Modal GPU Function (A10G / L4)             │
 │ ┌────────────────────────────────────────┐ │
 │ │ ffmpeg: decode → 16 kHz mono WAV       │ │
 │ │ (optional) Demucs: stem separation     │ │
 │ │ Basic Pitch / MT3: audio → MIDI        │ │
 │ │ music21: MIDI → MusicXML               │ │
 │ │ Push status updates → Postgres/Redis   │ │
 │ │ Upload MusicXML + MIDI → R2            │ │
 │ └────────────────────────────────────────┘ │
 └──────────────┬─────────────────────────────┘
                │ status writes
                ▼
 ┌────────────────────────────┐
 │ Postgres (Neon)            │
 │  jobs(id, user, status,    │
 │       r2_audio, r2_score)  │
 └────────────────────────────┘

Status changes are read by FastAPI's SSE endpoint (LISTEN/NOTIFY or
short-poll Redis pubsub) and streamed to the browser. When status=done,
the frontend fetches MusicXML via signed R2 URL and hands it to OSMD.
```

## Alternatives Considered

### Backend framework
- **FastAPI (chosen)** — Async by default, Pydantic validation, OpenAPI docs free, ~5x throughput vs Flask for I/O-bound endpoints, natural fit for Python ML ecosystem.
- **Flask** — Simpler but synchronous; workers block during long calls. Fine for tiny prototypes, not for SSE + ML orchestration.
- **BentoML** — Specialized ML serving with built-in batching. Worth revisiting post-MVP if we self-host inference; overkill when Modal owns the model runtime.
- **Node/Next.js API routes** — Rejected: Python is where the ML ecosystem (librosa, music21, Basic Pitch, MT3) lives.

### ML hosting
- **Modal (chosen)** — Sub-5s cold starts, Python-native SDK, automatic containerization, scale-to-zero. Best DX for a small team. ~$4.76/hr H100 (we'll mostly use cheaper A10G/L4).
- **Replicate** — Easiest to publish a public model, but custom-model cold starts are 11–60s and per-call pricing is higher. Good for hosting a public demo only.
- **RunPod Serverless** — Cheapest H100 (~$4.47/hr) and very fast cold starts (FlashBoot <200ms for 48% of starts), but BYO container ergonomics are rougher than Modal's Python decorators.
- **Beam** — Fast (<200ms) cold starts, good DX, smaller ecosystem. Strong fallback if Modal pricing becomes an issue.
- **fal.ai** — Excellent for Stable-Diffusion-style image/video; less proven for custom audio models.
- **AWS SageMaker / Vertex AI** — Production-grade but heavyweight; long setup, weak DX for indie/MVP velocity, no scale-to-zero on real-time endpoints.
- **Banana** — Effectively defunct/pivoted; not recommended.

### Frontend / mobile
- **Web-first PWA (chosen)** — One codebase, Web Audio + MediaRecorder is now solid on iOS 17+/Android, instant install, no app-store gating. A PWA can shave 30–40% of build cost vs hybrid and more vs dual native.
- **React Native** — Strong if we later need background audio capture or store distribution. New Architecture (0.76+) closes the perf gap.
- **Flutter** — Larger community share (46% vs RN 35% in 2026 surveys); CanvasKit web bundle now <800 KB. But sheet-music rendering ecosystem is JS-centric (OSMD/VexFlow), so React/web wins for the MVP.
- **Native iOS/Android** — Reserved for a v2 if we need CoreAudio low-latency live transcription.

### Sheet music renderer
- **OSMD (chosen)** — Renders MusicXML directly, built on VexFlow, actively maintained by PhonicScore. Our pipeline outputs MusicXML, so this is plug-and-play.
- **VexFlow alone** — Lower-level; would require us to lay out every measure manually. Too much frontend work for the MVP.
- **abcjs** — Good for ABC notation only; we use MusicXML.
- **Verovio** — Excellent quality and MEI/MusicXML support; heavier and slower to integrate. Reconsider for engraving-quality PDFs later.

### Audio pipeline
- **ffmpeg (chosen for I/O)** — Decode, resample, normalize. Universal.
- **librosa (chosen for features)** — Mature, NumPy-based.
- **torchaudio** — In maintenance phase as of 2.8/2.9; some user-facing features removed. Use only when we need GPU tensors directly.

### Job queue
- **Modal-native (chosen)** — `Function.spawn`, retries, autoscaling are built in. Removes Redis + worker fleet from the MVP critical path.
- **Celery + Redis** — Industry standard, more features (routing, schedules, chords). Add when we need DAGs or CPU-only side jobs.
- **RQ** — Simpler than Celery, Redis-only. Reasonable middle ground if we move off Modal-native.
- **Sidekiq** — Ruby-native; not relevant to a Python stack.

### Storage
- **Cloudflare R2 (chosen)** — Zero egress fees; ~$0.015/GB storage; S3-compatible. For media-serving SaaS, R2 is dramatically cheaper than S3 (cited example: 10 TB/mo egress = $15 on R2 vs $891 on S3).
- **AWS S3** — Better deep AWS integration; egress fees punish media workloads.
- **Supabase Storage / Backblaze B2** — Workable, smaller ecosystems.

### Database
- **Neon Postgres (chosen)** — Serverless, branchable, generous free tier, low cold-start.
- **Supabase Postgres** — Equally fine; choose if we adopt Supabase Auth too.
- **RDS** — Production-mature, but ops overhead and no scale-to-zero.

### Auth
- **Clerk (chosen)** — Best Next.js DX; auth working in <1 day; prebuilt UI; 10k MAU free; JWTs verifiable from FastAPI via JWKS.
- **Supabase Auth** — Best if we use Supabase DB; cheaper at scale ($0.00325/MAU after 50k vs Clerk's $0.02/MAU after 10k).
- **Auth0** — Enterprise SSO, expensive ($0.07/MAU). Overkill.
- **NextAuth/Auth.js** — Free, but you own MFA, social, sessions, rate limiting. Slower path to a polished MVP.

### Realtime
- **SSE (chosen)** — One-way progress events; works over plain HTTP; built-in reconnect with `Last-Event-ID`; firewall/proxy friendly. ~95% of "real-time" apps don't need WebSockets.
- **WebSockets** — Reserve for v2 features like live mic streaming with incremental notation.
- **Long polling** — Fallback only.

### Observability
- **Sentry (chosen)** — Errors, traces, releases, source maps; first-class Next.js + FastAPI SDKs.
- **OpenTelemetry + Grafana/Tempo** — Add as we scale; Sentry now has an OTLPIntegration that ingests OTel traces directly, so we can do both.

## Sources

- [Top Serverless GPU Clouds for 2026 (RunPod)](https://www.runpod.io/articles/guides/top-serverless-gpu-clouds)
- [Serverless GPU Platforms: RunPod, Modal, and Beam Compared (Introl)](https://introl.com/blog/serverless-gpu-platforms-runpod-modal-beam-comparison-guide-2025)
- [Best Serverless GPU Platforms for AI Apps and Inference in 2026 (Koyeb)](https://www.koyeb.com/blog/best-serverless-gpu-platforms-for-ai-apps-and-inference-in-2026)
- [Serverless GPU Hosting: Modal vs Replicate vs RunPod (Markaicode)](https://markaicode.com/serverless-gpu-modal-vs-replicate/)
- [10 Best Modal Alternatives in 2026 (Spheron)](https://www.spheron.network/blog/modal-alternatives/)
- [Modal Docs: Introduction](https://modal.com/docs/guide)
- [Modal Pricing](https://modal.com/pricing)
- [FastAPI vs Flask for Production AI APIs in 2026 (ClickIT)](https://www.clickittech.com/ai/fastapi-vs-flask-for-production-ai-apis/)
- [Breaking Up With Flask & FastAPI: ML Serving Frameworks (BentoML)](https://www.bentoml.com/blog/breaking-up-with-flask-amp-fastapi-why-ml-model-serving-requires-a-specialized-framework)
- [OpenSheetMusicDisplay GitHub](https://github.com/opensheetmusicdisplay/opensheetmusicdisplay)
- [List of Sheet Music Display Libraries for Browsers (OSMD)](https://opensheetmusicdisplay.org/blog/sheet-music-display-libraries-browsers/)
- [Exploring Music Transcription with Multi-Modal Language Models (Towards Data Science)](https://towardsdatascience.com/exploring-music-transcription-with-multi-modal-language-models-af352105db56/)
- [oh-sheet open-source pipeline (GitHub)](https://github.com/swifttarrow/oh-sheet)
- [Music Transcription with Transformers (Magenta)](https://magenta.tensorflow.org/transcription-with-transformers)
- [Flutter vs React Native: 2026 Market Share (Tech Insider)](https://tech-insider.org/flutter-vs-react-native-2026/)
- [Native vs Hybrid vs PWA in 2026 (DualMedia)](https://www.dualmedia.fr/en/native-hybrid-pwa-2026/)
- [PWA Setup Guide for Next.js 15 (DEV)](https://dev.to/rakibcloud/progressive-web-app-pwa-setup-guide-for-nextjs-15-complete-step-by-step-walkthrough-2b85)
- [What PWA Can Do Today: Audio Recording](https://whatpwacando.today/audio-recording/)
- [Choosing The Right Python Task Queue (Judoscale)](https://judoscale.com/blog/choose-python-task-queue)
- [Best infrastructure for Python AI backends and Celery workers in 2026 (Render)](https://render.com/articles/best-infrastructure-python-ai-celery-workers)
- [Cloudflare R2 vs AWS S3 (Cloudflare)](https://www.cloudflare.com/pg-cloudflare-r2-vs-aws-s3/)
- [Cloudflare R2 Pricing 2026 (LeanOps)](https://leanopstech.com/blog/cloudflare-r2-pricing-2026/)
- [Storage Wars: Cloudflare R2 vs Amazon S3 (Vantage)](https://www.vantage.sh/blog/cloudflare-r2-aws-s3-comparison)
- [Clerk vs Auth0 vs Supabase Auth 2026 (AppStackBuilder)](https://appstackbuilder.com/blog/clerk-vs-auth0-vs-supabase-auth)
- [Clerk vs Supabase Auth vs NextAuth (Better Dev / Medium)](https://medium.com/better-dev-nextjs-react/clerk-vs-supabase-auth-vs-nextauth-js-the-production-reality-nobody-tells-you-a4b8f0993e1b)
- [Clerk vs Auth0 vs Supabase Auth for Indie Hackers 2026 (DevToolPicks)](https://devtoolpicks.com/blog/clerk-vs-auth0-vs-supabase-auth-indie-hackers-2026)
- [SSE vs WebSockets 2026 (Nimbleway)](https://www.nimbleway.com/blog/server-sent-events-vs-websockets-what-is-the-difference-2026-guide)
- [Why SSE Beats WebSockets for 95% of Real-Time Apps (Medium)](https://medium.com/codetodeploy/why-server-sent-events-beat-websockets-for-95-of-real-time-cloud-applications-830eff5a1d7c)
- [SSE vs WebSockets (OneUptime)](https://oneuptime.com/blog/post/2026-01-27-sse-vs-websockets/view)
- [librosa GitHub](https://github.com/librosa/librosa)
- [torchaudio GitHub](https://github.com/pytorch/audio)
- [opentelemetry-instrumentation-fastapi (PyPI)](https://pypi.org/project/opentelemetry-instrumentation-fastapi/)
- [Sentry + OpenTelemetry Setups for Python (Medium)](https://medium.com/@Modexa/10-sentry-opentelemetry-setups-for-python-youll-reuse-forever-a3244f810c10)
- [Sentry Python OpenTelemetry Integration Docs](https://docs.sentry.io/platforms/python/tracing/instrumentation/opentelemetry/)
- [End-to-End LLM Observability in FastAPI with OpenTelemetry (freeCodeCamp)](https://www.freecodecamp.org/news/build-end-to-end-llm-observability-in-fastapi-with-opentelemetry/)
