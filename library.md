# Homelab Documentation Library

Central index of user-facing documentation on k3master.
For cluster operations, services, and deployments see the **k8s-ops skill** at `~/.claude/skills/k8s-ops/`.

Last updated: 2026-04-10

---

## Machine & Hardware

| File | Description |
|------|-------------|
| [info.md](info.md) | k3master hardware, OS, installed software, systemd services, cron jobs, automation scripts, SSH config. |
| [IPcams.md](IPcams.md) | IP camera hardware specs — Xiongmai, Reolink 4K, Yi cam. Stream resolutions, codecs, RTSP URLs, credentials, network isolation. |

## Credentials

| File | Description |
|------|-------------|
| [homelab-secrets/credentials.md](homelab-secrets/credentials.md) | All service credentials — Gitea, Grafana, WebUI, Kuma, SSH, cameras, Postgres. **Sensitive.** |

## Projects

| File | Description |
|------|-------------|
| [hydroflow/README.md](hydroflow/README.md) | HydroFlow IoT project — Express backend, Angular frontend, MQTT, PostgreSQL. |
| [hydroflow/CONTEXT.md](hydroflow/CONTEXT.md) | HydroFlow architecture decisions and deployment status. |
| [hydroflow/backend/handover.md](hydroflow/backend/handover.md) | Backend API routes, DB schema, environment variables. |
| [ivmos-guitar-practice-app/README.md](ivmos-guitar-practice-app/README.md) | Guitar practice app — deployed on k3master, default namespace. |
| [project_ideas.md](project_ideas.md) | Phone GPU compute project ideas — evaluated with hardware constraints (USB, thermal, memory). Reviewed by Claude + Gemini (3 rounds). |
| [blog-frigate-nvr.md](blog-frigate-nvr.md) | Blog post draft — running Frigate NVR on K3s with IP cameras, face recognition, Cloudflare tunnel. |
| [gpu-worker/README.md](gpu-worker/README.md) | GPU inference worker — YOLOv8 on ARM64 phone GPUs via ONNX Runtime + OpenCL. |

## Reference & Patterns

| File | Description |
|------|-------------|
| [sidecar-manifests/README.md](sidecar-manifests/README.md) | Fluent-bit sidecar logging pattern for guitar-backend → Loki. |
| [sidecar-manifests/blog-post-sidecar-pattern.md](sidecar-manifests/blog-post-sidecar-pattern.md) | Blog reference on the K8s sidecar logging pattern. |

## K8s Operations Skill (cluster reference)

The `~/.claude/skills/k8s-ops/` directory replaces the former `INFRA.md` as the cluster reference:

| File | Contents |
|------|----------|
| `SKILL.md` | Cluster topology, MetalLB IPs, namespaces, SOPs, constraints, known issues |
| `references/frigate.md` | Frigate NVR — cameras, config workflow, night tuning, backup, Cloudflare |
| `references/monitoring.md` | Prometheus, Alertmanager, PrometheusRules, Grafana dashboards, phone metrics |
| `references/phone-gpu.md` | Adreno 630 OpenCL specs, container GPU mount pattern, project roadmap |
| `references/network.md` | Physical topology, USB tethering, MetalLB L2 issue, CNI, Cloudflare tunnel |

## Manifest Directories

| Directory | Contents |
|-----------|----------|
| `~/frigate-k8s/` | Frigate NVR K8s manifests — write-back sidecar, RBAC, secret mgmt. GitHub: `ivemcfire/frigate-k8s` |
| `~/gpu-worker/` | GPU inference worker — worker.py, Dockerfiles, K8s manifests, air-gap stagers. GitHub: `ivemcfire/gpu-worker` |
| `~/monitoring-stack/` | Alertmanager, Frigate exporter, PrometheusRules, Grafana dashboard JSON |
| `~/hydroflow/INFRA/k8s/` | HydroFlow manifests (infrastructure, backend, frontend) |
| `~/homelab-config/apps/` | App manifests (gitea, open-webui, uptime-kuma, guitar) |
| `~/homelab-config/monitoring/` | Fluent-bit sidecar config |
