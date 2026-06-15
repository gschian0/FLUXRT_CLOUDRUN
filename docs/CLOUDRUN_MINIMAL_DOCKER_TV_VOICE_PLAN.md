# Cloud Run Minimal Docker TV + Voice Plan

## Goal
Build a production-ready, minimal-footprint system that:
- Runs a control API on Cloud Run.
- Runs GPU rendering workers outside Cloud Run (GCE GPU or GKE GPU).
- Executes a TV-style schedule (shows, segments, promos, breaks).
- Generates narration voice + background audio automatically.
- Publishes continuous streams to YouTube and Twitch with low operational overhead.

## Why Hybrid (Cloud Run + GPU Worker)
Cloud Run does not support NVIDIA GPUs for general workloads in this setup, so heavy inference stays on GPU workers.
Cloud Run is still ideal for:
- Scheduling and orchestration.
- Lightweight APIs and state management.
- Webhooks, health checks, and admin UI.

## Final Architecture
1. Cloud Run Control Plane:
- Scheduler API.
- Playlist compiler.
- Segment metadata service.
- TTS/audio job dispatcher.
- Worker lifecycle control.

2. GPU Worker Runtime:
- FluxRT render/inference pipeline.
- Overlay compositor.
- Final AV mux and RTMP publishing.
- Local watchdog + self-healing.

3. Storage and Messaging:
- Cloud Storage for assets, generated clips, and manifests.
- Firestore or Cloud SQL for schedule/state.
- Pub/Sub for job dispatch and status events.

4. Secrets and Security:
- Secret Manager for stream keys and HF token.
- Service accounts with least privilege.

## Minimal Container Strategy

### Control Plane Image (Cloud Run)
Use a very small image:
- Base: python:3.11-slim (or distroless for final hardening).
- Install only API dependencies (FastAPI, uvicorn, google-cloud libs).
- No CUDA, no OpenCV-heavy stack, no FFmpeg build chain if not needed in control plane.

Target characteristics:
- Image size: ~250MB or less.
- Fast cold starts.
- Horizontal scaling on request load.

### GPU Worker Image (GCE/GKE)
Use one dedicated worker image:
- Base: nvidia/cuda runtime matching host driver.
- Install FFmpeg, PyTorch CUDA wheel, FluxRT deps, audio stack.
- Bundle only runtime scripts (no dev/test tools).

Target characteristics:
- Reproducible startup.
- Prewarmed model cache support.
- Strong health endpoints.

## Make Install Footprint Small
1. Split dependencies:
- requirements-control.txt (Cloud Run only).
- requirements-worker.txt (GPU runtime only).

2. Remove unnecessary packages from control plane:
- No torch, no heavy CV libs, no model packages.

3. Use multi-stage Docker builds:
- Build wheels in builder stage.
- Copy only wheels/site-packages + app into final stage.

4. Enable .dockerignore aggressively:
- Exclude model files, sample videos, notebooks, git metadata, logs, caches.

5. Keep models out of image:
- Download on worker startup only when missing.
- Optionally pre-seed a persistent disk cache.

## Hugging Face Token and Fast Model Pull
1. Store HF token in Secret Manager.
2. Inject as HF_TOKEN env var into worker runtime.
3. On startup:
- Authenticate once.
- Pull model snapshots into cache dir.
- Reuse cache on restart.

Recommended env:
- HF_TOKEN
- HF_HOME=/models/hf
- TRANSFORMERS_CACHE=/models/hf/transformers
- HUGGINGFACE_HUB_CACHE=/models/hf/hub

Startup optimization:
- Worker boot script checks local cache hash first.
- Only missing artifacts are downloaded.

## TV Schedule System Design

### Data Model
- Show: title, theme, prompt profile, voice style, duration.
- Segment: intro, main scene, interstitial, outro.
- Playlist block: ordered segments with UTC timestamps.
- Break: bumper/ad/idle loop with fallback narration.

### Scheduler Flow
1. Cron job builds next 24h schedule.
2. Compiler generates segment manifest JSON.
3. Manifest is sent to active worker via Pub/Sub.
4. Worker acknowledges each segment and sends heartbeat.

### Runtime Guarantees
- Always have next N segments queued.
- Fallback segment when generation fails.
- Automatic rollover to standby worker on missed heartbeat.

## Generated Voice and Sound

### Voice (TTS)
- Use cloud TTS or local TTS model service.
- Generate narration from segment script templates.
- Save WAV files to storage and normalize loudness.

### Background Audio
- Use music generator pipeline or pre-curated loops.
- Keep stems per mood/theme.
- Crossfade between segments.

### Final Audio Bus
- Mix: narration + background + optional SFX.
- Apply limiter and loudness target (e.g., -14 LUFS streaming target).
- Feed AAC stereo into final mux.

## End-to-End Stream Path
1. Source scene rendered on GPU worker.
2. Overlay/title package composited.
3. Narration/music mixed.
4. FFmpeg outputs stable H.264 + AAC RTMP.
5. Tee fanout pushes to YouTube and Twitch.

## Reliability and Auto-Recovery
1. Health checks:
- Control plane liveness/readiness.
- Worker heartbeat every few seconds.
- Stream output monitor (fps, bitrate, dropped frames).

2. Restart policy:
- Restart fanout on ingest failure.
- Restart worker on repeated model/render failures.

3. Failover:
- Keep warm standby worker.
- Control plane reassigns schedule cursor automatically.

## Cloud Run Service Configuration
- Min instances: 1 for warm API.
- Max instances: based on API traffic.
- CPU always allocated: enabled for scheduler responsiveness.
- Request timeout: long enough for orchestration calls.
- Concurrency: tune for API workload (not rendering).

## CI/CD Plan
1. GitHub Actions:
- Lint/test.
- Build control image.
- Build worker image.
- Push to Artifact Registry.

2. Deploy stages:
- Dev
- Staging
- Production

3. Promotion gates:
- Smoke tests for schedule compile.
- Worker startup and model cache validation.
- RTMP ingest test to private endpoints.

## Cost and Performance Controls
1. Autoscale worker count by schedule demand.
2. Turn off idle workers outside programmed windows.
3. Use persistent disks for model cache reuse.
4. Keep one warm worker before top-of-hour transitions.

## Security Baseline
1. All keys in Secret Manager.
2. Signed images and pinned digests.
3. Private VPC where possible.
4. IAM role separation for control vs worker.

## Phased Execution Plan

### Phase 1: Container Foundation (1-2 days)
- Split dependencies into control and worker sets.
- Add Dockerfiles + docker-compose for local parity.
- Add startup scripts and health endpoints.

### Phase 2: Control Plane (2-3 days)
- Build scheduler API and manifest compiler.
- Add Firestore/SQL schedule state.
- Add Pub/Sub dispatch.

### Phase 3: Worker Runtime (2-4 days)
- Integrate render + overlay + audio mix + fanout.
- Add HF token-driven model bootstrap cache.
- Add watchdogs and self-healing restarts.

### Phase 4: TV Schedule + Voice (3-5 days)
- Implement show templates and segment script generation.
- Add TTS generation pipeline.
- Add background music flow and loudness normalization.

### Phase 5: Production Hardening (2-3 days)
- Add dashboards/alerts.
- Add failover worker.
- Load test and latency tuning.

## Definition of Done
- One command deploy for control plane to Cloud Run.
- Worker can start from clean VM, pull models quickly with HF token, and stream within target startup window.
- 24h scheduled programming runs without manual intervention.
- YouTube and Twitch both receive stable H.264/AAC with no sustained ingest warnings.

## First Implementation Tasks (Immediate)
1. Create requirements-control.txt and requirements-worker.txt.
2. Add Dockerfile.control and Dockerfile.worker.
3. Add worker bootstrap.sh with model cache + HF token checks.
4. Add schedule manifest schema and sample day schedule.
5. Add control API endpoints: compile schedule, dispatch segment, worker heartbeat.
