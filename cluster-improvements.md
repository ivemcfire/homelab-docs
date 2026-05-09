# Cluster improvements registry

Running list of cluster issues, tech-debt items, and opportunities discovered during the **DevOps tutor track** (and other reviews). Single source of truth — append, don't fork.

Indexed from `~/.claude/library.md`. Source-of-truth path: `user@192.168.100.52:~/homelab-docs/cluster-improvements.md`.

**Severity:** L (low / cosmetic) · M (medium / paper cut) · H (high / actively risky).
**Status:** open · in-progress · deferred · tracked-elsewhere · done.

---

## Open items

| # | Date found | Sev | Topic | Finding | Proposed fix | Status |
|---|-----------|----:|-------|---------|--------------|--------|
| 1 | 2026-05-04 | H | k8s version skew | Phones (one61/62/6t) at k3s v1.35.0; k3master + k3frigate at v1.34.3. Reversed skew (workers ahead of control plane). | Upgrade control plane to v1.35.x **before** any further worker change. Order: master -> k3frigate -> phones (already done). Verify with `kubectl get nodes` post-upgrade. | open |
| 2 | 2026-05-04 | M | Phone-to-cluster network split-brain | Cross-node pod traffic to phone Pod IPs from k3master/k3frigate broken since 2026-04-29 node-ip flip. | Already documented in `~/.claude/skills/k8s-ops/references/network.md`. Resolve as part of Phase 2 HA rollout (per `phase2-plan.md`). | tracked-elsewhere |
| 3 | 2026-05-04 | M | No role labels on phones | one61/62/6t have only built-in labels. App placement is by chance, not policy. | Add `homelab/role=mqtt|api|frontend` labels per node; add `nodeSelector` to relevant Deployments after labels stabilize. Reversible. | open |
| 4 | 2026-05-04 | M | `default` namespace is a grab-bag | gitea, loki, uptime-kuma, open-webui, guitar-backend/frontend all live in `default`. No quotas, no clear ownership. | Migrate each to a dedicated namespace (`gitea`, `logging`, `uptime-kuma`, `open-webui`, `guitar`). Move incrementally; update Service DNS references. | open |
| 5 | 2026-05-04 | M | gpu-inference worker not on GPU node | `mqtt-inference-worker-*` Pod runs on `one62` (no GPU). The `nvidia.com/gpu.present=true` label is on `k3frigate`. | Add `nodeSelector: nvidia.com/gpu.present: "true"` (or proper resource request `nvidia.com/gpu: 1`) to Deployment. Verify Pod re-schedules to k3frigate. | open |
| 6 | 2026-05-04 | L | Stale Errored Pods cluttering listings | `open-webui-7cc6bcffd4-98m7d` (Error 60d, no IP), `frigate-backup-test-1777460770-*` (Error 4d17h), `opencl-test` (Error 24d). | One-time cleanup with `kubectl delete pod ...` once confirmed stale. Add CronJob or `ttlSecondsAfterFinished` for future Job runs. | open |
| 7 | 2026-05-04 | M | Single control plane = SPOF | k3master is the only control plane. Loss = no scheduling, no kubectl until rebuild. | Phase 2 plan (`phase2-plan.md`) targets 3-node etcd HA. SP4 (`k3node2`) not yet provisioned. | tracked-elsewhere |
| 8 | 2026-05-04 | L | Control plane runs app workloads | `chickenflow-backend` deliberately pinned to k3master via `chickenflow/storage=true`. Acceptable for homelab; in production CPs are tainted. | Document as deliberate deviation; revisit if/when CP is re-tainted during HA rollout. | deferred |
| 9 | 2026-05-04 | M | Multi-arch image discipline | Phones are `arm64`, k3master/k3frigate are `amd64`. Any single-arch image silently fails to schedule on phones. | Build pipeline must produce multi-arch manifests (`docker buildx build --platform=linux/amd64,linux/arm64 ...`). Document as a CI rule once CI exists. | open |
| 10 | 2026-05-04 | L | GardenFlow not deployed | No `gardenflow` namespace in cluster. Local frontend points at `192.168.100.208:5000` which is one62 LAN IP -- currently broken given network split. | Plan GardenFlow deployment as a tutor-track milestone: containerize -> registry -> manifests -> namespace. Will exercise the whole pipeline. | open |
| 11 | 2026-05-04 | M | LoadBalancer stuck on `<pending>` | `default/guitar-backend` Service is type=LoadBalancer but EXTERNAL-IP shows `<pending>` (77d). Either MetalLB pool is exhausted/misconfigured for this Service, or it duplicates `guitar-frontend` which already has `.202`. | Inspect with `kubectl describe svc guitar-backend` + `kubectl -n metallb-system get ipaddresspools`. Either assign a pool, change to ClusterIP if internal-only, or delete if redundant. | open |

---

## Closed items

---

## 2026-05-09 session findings (face-recognition + switch deploy + production polish)

| # | Date found | Sev | Topic | Finding | Proposed fix | Status |
|---|-----------|----:|-------|---------|--------------|--------|
| 12 | 2026-05-09 | H | Frigate MQTT silently broken cross-namespace | Frigate's `mqtt.host` pointed at `mosquitto.hydroflow.svc.cluster.local` (one6t pod IP). Cross-node Pod traffic from k3frigate to phone Pod IPs broken since 2026-04-29. Frigate published nothing for 10+ days; no log indicated failure. | Resolved: deployed local mosquitto in `frigate` ns pinned to k3frigate; repointed Frigate + Double Take. Underlying split-brain still tracked-elsewhere (#2). | done |
| 13 | 2026-05-09 | H | Frigate native face_recognition non-functional | Frigate 0.17 face_rec required exact frontal face capture. Sideways walks past cam17/cam22 = silent miss. Stack also failed because `/media/frigate/clips/faces/` was empty post-PVC migration to k3frigate. | Replaced with Double Take + CompreFace stack on k3frigate (CPU-only). Frigate native face_rec disabled globally + per-cam. | done |
| 14 | 2026-05-09 | M | k3master disk pressure | Root partition was 82% full (76 GB / 98 GB). k3s containerd cache held 23 GB of stale image layers. | Pruned via `crictl rmi --prune` — reclaimed 8 GB (now 73%). Recommend monthly cron prune. /home/user 18 GB still untouched (user review). | done (one-shot) |
| 15 | 2026-05-09 | L | Frigate /dev/shm too small for 1080p detect | Warning `SHM size of 256MB is too small, recommend ≥262MB` after bump to 1080p detect resolution on cam17 + cam22. | Bumped emptyDir sizeLimit 256Mi → 512Mi. | done |
| 16 | 2026-05-09 | M | Frigate OOMKilled after cam22 add | 5MP main stream + ArcFace large model + cam17 1080p detect saturated 3 Gi memory limit. | Bumped `resources.limits.memory` 3Gi → 5Gi initially, then trimmed to 4Gi after ArcFace strip + native face_rec disable. Steady ~3 Gi peak. | done |
| 17 | 2026-05-09 | M | CompreFace empty PVC shadowed bundled models | Mounting empty `compreface-models` PVC at `/app/ml/.models` hid the model files baked into the image. `facenet.Calculator` failed on `20180402-114759.pb not found`. | Removed PVC mount entirely; image-bundled models load fine. PVC + PV deleted. Pattern: only mount PVCs over data dirs, not over image-provided assets. | done |
| 18 | 2026-05-09 | L | CompreFace fe nginx env name + value format | `CLIENT_MAX_BODY_SIZE` env required (not `MAX_FILE_SIZE`); value must be nginx-compliant `10M` not `10MB`. Took 4 crashloops to land. | Documented in face_recognition.md. Required envs: `CLIENT_MAX_BODY_SIZE`, `PROXY_READ_TIMEOUT`, `PROXY_CONNECT_TIMEOUT`. | done |
| 19 | 2026-05-09 | M | CompreFace core OOMKilled at 2 Gi | TensorFlow + 7 plugins (face detector + landmarks + age + gender + mask + pose + calculator) saturate 2 Gi. | Bumped core mem limit 2Gi → 4Gi. Steady ~1.7 Gi after warmup. | done |
| 20 | 2026-05-09 | M | MetalLB Memberlist UDP probes failing | `Was able to connect to k3frigate over TCP but UDP probes failed, network may be misconfigured` — chronic warn flood between k3master ↔ k3frigate. Speaker still functional via TCP gossip. | Investigate WireGuard mesh UDP rules. Separate networking ticket. | open |
| 21 | 2026-05-09 | L | Jumphost dual-IP binding | Same MAC (`b8:70:f4:c6:60:70`) answers ARP on both `.62` (V17.2 official) AND `.152` (legacy). | Remove `.152` binding from jumphost iface (netplan or `/etc/network/interfaces`), restart networking. | open |
| 22 | 2026-05-09 | L | Suspicious locally-admin MACs on LAN | arp-scan showed devices at `.41/.51/.133/.140/.179` with locally-administered MAC bits set. | Inventory before VLAN segmentation — mostly likely chromecasts/IoT/random-MAC clients. | open |
| 23 | 2026-05-09 | M | cam22 NTP not syncing | Cam GUI hardening (NTP→`192.168.100.52`) didn't apply — embedded timestamp `03/05/2026` vs actual `09/05/2026`. | Re-set NTP via cam GUI using raw IP (DNS nulled per hardening). Verify drift. | open |
| 24 | 2026-05-09 | L | Liveness probes missing on face-rec stack | New deployments (compreface-{core,admin,api,fe}, postgres, double-take, mosquitto) had no liveness probe — stuck pods would not auto-recover. | Added TCP-socket liveness probes (initialDelay 60–180s, periodSeconds 30, failureThreshold 3) on all 7 deployments. | done |
| 25 | 2026-05-09 | L | CompreFace postgres lacked backup strategy | Training data + face library = irreplaceable; no dump strategy. | Deployed CronJob in `face-recognition` ns: daily 3am `pg_dump` + gzip to hostPath `/mnt/frigate/compreface/postgres-backups`, 14-day retention. Manually tested. | done |
| 26 | 2026-05-09 | L | Frigate cam15 chronic RTSP outage | `dial tcp 192.168.100.15:554: i/o timeout` for 20+ hours; cam likely offline (power/network). | Verify cam15 power/PoE/network. Not Frigate side. | open |
| 27 | 2026-05-09 | L | Frigate cam19_yicam codec parse fail | yi-hack stream had AAC audio + h264 video; ffmpeg couldn't lock codec params on sub-stream. | Switched cam19 to single main stream input with `-an` (drop audio) + bigger probesize. cam19 recording at 7 fps now. | done |
| 28 | 2026-05-09 | L | Frigate cam17 4K HEVC mislabeled | go2rtc URL `h264Preview_01_main` actually delivers H.265 because cam set to HEVC. Cosmetic — NVDEC handles both. | Update Frigate config comment for cam17 stream codec note. | open |


---

## 2026-05-09 follow-up (fix pass on open items)

| # | Update |
|---|--------|
| 3 | done — applied `homelab/role` labels: k3master=control-plane, k3frigate=video, one6t=mqtt, one62=db, one61=app. No workloads pinned via these yet (advisory). |
| 5 | reclassified — phone Adreno GPU is the intended target (arm64 image, OpenCL). Item was mistaken about k3frigate as target. Logs show last detection 2026-04-27 (12d stale) — pod alive but worker stuck. Separate ticket. |
| 6 | done — deleted errored pods (open-webui-...-98m7d, opencl-test, frigate-backup-test) + 3 failed Jobs. Frigate-backup CronJob has `successfulJobsHistoryLimit/failedJobsHistoryLimit` already; no recurrence expected. |
| 11 | partial — corrected pool annotation `default` -> `homelab-pool`. Real cause: shared-IP collision with guitar-frontend on .202. Runbook saved at `~/homelab-docs/lb-guitar-backend-fix.md` with 3 fix paths (ClusterIP / different IP / shared-IP annotation). User decision required. |
| 20 | partial — rotated MetalLB memberlist secret key (was version-88 mismatch causing decryption failures since k3s 1.34/1.35 skew). Speakers restarted clean. Residual UDP probe warnings remain due to phone-LAN split-brain (same root as #2) — TCP gossip works, MetalLB functional, treat as cosmetic until cross-node Pod-IP routing fixed. |
| 21 | runbook saved — `~/homelab-docs/jumphost-fix-runbook.md`. Sudo on .62 needs interactive pwd; user runs when ready. |
| 28 | done — Frigate config comment updated to clarify Reolink URL says `h264Preview` but delivers actual cam codec (H.265 in our config). |
