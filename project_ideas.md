# Phone Worker Project Ideas — Final

Date: 2026-04-08
Reviewed by: Claude + Gemini (3 revision rounds)

---

## Hardware Reality

### Per Phone (x3: one61, one62, one6t)

| Component | Spec | Status |
|-----------|------|--------|
| CPU | 4x Kryo 385 Gold 2.8GHz + 4x Silver 1.8GHz | Idle (~2%) |
| GPU | Adreno 630 (OpenCL via Mesa Rusticl) | Idle, compute-capable |
| RAM | 6GB (one61, one62) / 8GB (one6t) | ~4.5GB / ~6.5GB usable after OS+K3s |
| Storage | UFS 2.1 | Not suited for sustained write I/O |
| DSP | Hexagon 685 | Inaccessible (proprietary firmware) |
| OS | postmarketOS v25.12 (Alpine/musl) | Mesa Rusticl in repos |

### Cluster Totals
- CPU: 24 cores (mostly idle)
- GPU: ~1.5 TFLOPS FP32 (3x Adreno 630)
- RAM: ~15.5GB usable
- Unified memory: CPU/GPU share RAM — no PCIe copy overhead

### Hard Constraints

**USB 2.0 Hub (all phones share one upstream link to k3master):**
- Shared bandwidth: ~30-40 MB/s real-world across all 3 phones
- Per phone: ~10-13 MB/s
- Good for: small request/response (detection frames, API calls, text tokens)
- Bad for: bulk transfers (video files, container images, storage replication)

**Thermal (no active cooling):**
- Snapdragon 845 aggressively throttles under sustained GPU load
- Continuous inference WILL hit thermal limits within minutes
- Frigate's motion filter is NOT a reliable thermal safeguard — environmental
  triggers (wind, rain, shadows) cause continuous motion events

**Memory:**
- 4.5GB usable on 6GB phones after OS + K3s + container runtime
- GPU allocations come from the same pool — no separate VRAM
- OOM kills from GPU workloads will take down the K3s agent

**Flash:**
- UFS 2.1 — fast reads, but sustained random writes degrade lifespan
- Not suitable for CI/CD builds or database-heavy write workloads

---

## Viable Projects

### 1. GPU-Accelerated Video Analytics Pipeline

**Priority: #1 — Best overall fit for this hardware.**

**What:** Event-driven secondary analysis of Frigate detection snapshots on phone GPUs.

**Why it's the best fit:**
- Event-driven: GPU spins up for a few seconds per event, then idles. No thermal issue.
- Low bandwidth: one snapshot (~50-100KB) per person detection, a few times per hour.
- Real utility: adds capabilities Frigate doesn't have natively.
- Builds on existing infra: Mosquitto MQTT, PostgreSQL, Frigate events, Grafana.

**Use cases (pick one or more):**
- License plate recognition (ALPR) on driveway cameras
- Enhanced face recognition — larger GPU-accelerated model vs Frigate's small CPU model
- Scene classification — "package at door", "car arrived", "gate left open"
- Animal/pet detection — secondary model specialized beyond Frigate's "person" tracking

**Architecture:**
```
Frigate detects person
    → publishes MQTT event to mosquitto.hydroflow.svc
        → Analytics worker on phone GPU subscribes
            → downloads snapshot (~50-100KB)
            → runs secondary model via OpenCL
            → publishes enriched result to MQTT
            → stores in PostgreSQL (on one62)
                → Grafana dashboard + Telegram notification
```

**Stack:**
- ONNX Runtime with OpenCL Execution Provider (or OpenCV DNN + OpenCL backend)
- Paho MQTT client (Python)
- PostgreSQL (already running on one62)
- K8s Deployment with /dev/dri device access
- Mesa Rusticl for OpenCL

**Container build (the hard part):**
Building OpenCV + ONNX Runtime with OpenCL on Alpine/musl/ARM64 is a dependency
nightmare. Expect:
- musl vs glibc linking issues (ONNX RT assumes glibc in many places)
- Mesa Rusticl pulls in Rust toolchain + LLVM as build deps
- Minimal community docs for this exact combination
- Multi-stage Dockerfile, long build times
- Budget 2-3 days for the container image alone

**Mitigation:** Consider using a Debian-slim base instead of Alpine for the analytics
container. The phones run Alpine, but K8s containers are isolated — a glibc-based image
works fine on a musl host. This sidesteps the musl compilation issues at the cost of a
larger image (~300MB vs ~100MB). The one-time image pull over USB 2.0 is slow but
acceptable.

**Difficulty:** Hard (container build), Medium (application logic).

**Resume line:** "Built event-driven GPU-accelerated video analytics pipeline on ARM64
edge nodes, processing NVR events via MQTT with secondary ML classification"

---

### 2. Frigate Object Detection Offload

**Priority: #2 — Real value, but less urgent than it sounds.**

**What:** Move Frigate's object detection from k3master CPU to phone GPUs via HTTP
detector API.

**Honest assessment:**
- The 82% CPU stat is the ENTIRE Frigate container (go2rtc, recording, ffmpeg, nginx,
  detection). The detector alone is a fraction of that.
- Current CPU inference at 28ms is healthy — target is <100ms.
- The real win is headroom for enabling cam04/05/06 and adding future cameras.
- This is NOT an urgent fix — it's capacity planning.

**Architecture:**
```
Cameras → Frigate (k3master) → HTTP detection request → Phone GPU (round-robin)
                                                         one61: detector:8080
                                                         one62: detector:8080
                                                         one6t: detector:8080
```

**USB 2.0 impact:** Minimal. Detection frames are ~30KB JPEG (320x320).
At 25 FPS = ~750 KB/s total. Well within budget.

**Thermal mitigation (CRITICAL):**
Frigate's motion filter is not sufficient as a thermal safeguard. Environmental
triggers (wind in trees, rain, shadows, lighting changes) cause continuous motion
events that would run the GPU non-stop.

**Mandatory:** Implement a hard server-side rate limit at the detection API level
(e.g. max 10 req/sec per phone), independent of Frigate's motion triggers.
This is a cap on the detector service itself, not relying on Frigate to be
well-behaved. Additionally:
- Monitor GPU temperature via sysfs, expose as Prometheus metric
- Alert when approaching throttle threshold (~85C)
- Consider physical cooling (120mm USB fan strapped to phones)

**Shares container build with Project 1** — same ONNX/OpenCL/Mesa stack. Build once,
deploy for both use cases.

**Difficulty:** Hard (same container build as #1), Medium (application logic).

**Resume line:** "Distributed GPU-accelerated object detection across ARM64 edge nodes
with rate-limited thermal management and Prometheus observability"

---

### 3. Edge LLM Inference (optional — interview flex only)

**Priority: #3 — Fun project, low practical value.**

**What:** Run small LLMs on phone GPUs via Tinygrad's OpenCL backend, serve via
OpenAI-compatible API to open-webui.

**Honest assessment:**
- You already have Ollama on the Windows PC with more RAM and proper cooling.
- The phone LLM will be slower, smaller, and thermally limited.
- The only value is the interview story: "I run LLM inference on recycled phones."
- That story IS genuinely memorable and unique — just don't pretend it's practical.

**Memory budget (hard limits):**
- one61/one62 (6GB): ~4.5GB usable → INT4 mandatory
  - TinyLlama 1.1B INT4: ~600MB weights + KV cache = fits
  - Phi-2 2.7B INT4: ~1.5GB weights + KV cache = tight, risk of OOM
- one6t (8GB): ~6.5GB usable → more headroom but still INT4
- KV cache grows with context window — must limit max context to prevent OOM
- MUST set K8s memory limits to protect K3s agent from being OOM-killed

**Thermal:** LLM inference is continuous GPU load during generation. Tokens/sec will
degrade as the phone throttles. Acceptable for a demo, not for production use.

**Stack:** Tinygrad, OpenCL, Mesa Rusticl, FastAPI, K8s Deployment

**Difficulty:** Medium-high. Tinygrad API is less stable. Quantization tuning needed.

**Resume line:** "Built OpenAI-compatible LLM serving cluster on recycled ARM64 mobile
devices with GPU inference via OpenCL"

---

## Not Viable (killed with justification)

| Project | Reason |
|---------|--------|
| CI/CD Runners | Unwinnable RAM/flash trade-off. tmpfs builds OOM-kill K3s agent (4.5GB usable, compilations need 2-4GB). Flash builds degrade UFS 2.1 lifespan. |
| GPU Transcoding | USB 2.0 transfer time (~20s per file) dominates GPU transcode time (~2s). k3master Intel QSV already handles encoding. Net speedup is marginal. Also causes USB bus contention with detection traffic. |
| MinIO Storage | Distributed storage needs fast inter-node replication. 10 MB/s per phone over shared USB 2.0 makes this pointless. |
| Redis Cluster | Technically viable but low value. HydroFlow isn't deployed yet, and a single Redis instance would suffice when it is. 3-node cluster is resume padding without real utility. |
| Log Pipeline | Technically viable but premature. Loki issues (Pending chunks-cache) should be fixed on k3master first. Moving Loki to phones adds complexity without solving the root cause. |

---

## Implementation Order

```
Phase 1: Container image (shared by Projects 1 & 2)
         Build ARM64 container with Mesa Rusticl + ONNX Runtime OpenCL EP
         This is the hardest and highest-risk step. If this fails, both
         GPU projects are blocked. Prototype on one phone first.

Phase 2: Project 1 — Video Analytics Pipeline
         MQTT subscriber + secondary model + PostgreSQL storage + Grafana
         Start with one use case (ALPR or face recognition), expand later.

Phase 3: Project 2 — Frigate Detection Offload
         HTTP detector service + rate limiter + thermal monitoring
         Reuses the same container image from Phase 1.

Phase 4 (optional): Project 3 — LLM Inference
         Tinygrad + OpenCL + FastAPI
         Different stack, do this only if you want the interview story.
```

**The critical path is the container image build.** If ONNX Runtime + OpenCL compiles
cleanly on ARM64 with Mesa Rusticl, everything else is application logic. If it doesn't,
evaluate Tinygrad as an alternative ML runtime (proven on this hardware) or fall back to
CPU-only inference (still faster than nothing, using the idle ARM cores).
