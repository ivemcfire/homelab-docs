# Master Network & Cluster Architecture Document V17.4

Status: Phase 3 progressing. Service room switch live; Attic + Inverter benched. Face Recognition Stack live + tuned (crop=1 + face cams only + face library purged for clean retrain). Cross-namespace MQTT split-brain fixed via static routes. HydroFlow postgres migrated off phone to k3frigate.

Last updated: 2026-05-10. Supersedes V17.3 (id `1k9hM1Hp4-BpP2BeAejpuj-hCo5rWY9pmez98Jx09Erw`).

## 1. Physical Infrastructure & Network Core

- Gateway: Xiaomi BE6500 (192.168.100.1). Locked. OTA cron stripped, DNS sinkhole, INITTED_OTA=1. xmir-patcher SSH unlock only — NOT full OpenWrt; mesh kept; no full firewall ruleset.

### Switching (ZyXEL GS1200-8 v3)

| Hostname | Mgmt IP | MAC | Status |
|----------|---------|-----|--------|
| zyxel-service | 192.168.100.2 | c0:2e:5f:57:fe:d4 | LIVE service room (8/8 ports used) |
| zyxel-attic | 192.168.100.3 | c0:2e:5f:58:01:7f | Benched, placement pending |
| zyxel-inverter | 192.168.100.4 | c0:2e:5f:58:01:79 | Benched, placement pending |

All 3 hardened: HTTPS only, HTTP/SNMPv2c-write/Telnet off, EEE off, Loop Guard + IGMP Snooping on, SNMPv2c read with per-switch random Get community.

## 2. k3s Cluster Topology

| Hostname | Hardware | IP | Role |
|----------|----------|-----|------|
| k3master | IdeaPad Flex 5 (i5-1035G1) | 192.168.100.52 | Server / Control plane / Chrony NTP for cams |
| k3frigate | i5-6600 + GTX 1050Ti | 192.168.100.56 | Agent — Frigate compute + face-recognition + hydroflow postgres |
| LocalAI | Win10, i5-10400F + RTX 3060 12GB | 192.168.100.50 | Local AI worker (Ollama) |
| Jumphost | (admin host) | 192.168.100.62 | Network management + backup target. Dual-IP cleanup done 2026-05-10. |
| acergo16 | Fedora laptop | 192.168.100.65 | Admin station |

Phones one61/62/6t (10.0.x.2 USB-tether). Hydroflow remaining: mosquitto + hydroflow-backend (one6t), gitea. HydroFlow postgres MIGRATED to k3frigate 2026-05-10.

### Split-brain — RESOLVED 2026-05-10

Static routes on k3frigate via netplan:

```yaml
- to: 10.0.1.0/30
  via: 192.168.100.52
- to: 10.0.2.0/30
  via: 192.168.100.52
- to: 10.0.3.0/30
  via: 192.168.100.52
```

VXLAN flow restored. Phone-hosted services reachable from frigate-namespace pods.

### MetalLB

L2 advertisement on `wan0` (k3master). Pool `homelab-pool` 192.168.100.200-220. Memberlist key rotated 2026-05-10 (was k3s 1.34/1.35 mismatch).

## 3. Frigate NVR + Camera Inventory

Software: ghcr.io/blakeblackshear/frigate:0.17.1-tensorrt
Compute: k3frigate (1050Ti, 4GB VRAM)
Mem limit: 4Gi. /dev/shm: 512Mi.
NTP server for cams: k3master 192.168.100.52.
Hardening per cam: null gateway, null DNS, NTP=192.168.100.52, P2P/Cloud disabled.

### Camera IP allocation (192.168.100.11-29)

| IP | Cam | Model | Codec | Role |
|----|-----|-------|-------|------|
| .11-.13, .15, .16 | cam11-16 (XMeye) | NETSurveillance | H.264 | object detection only |
| .17 | cam17_reolink | Reolink 4K | H.265 | object detection + face recognition |
| .19 | cam19_yicam | Xiaomi Yi (yi-hack) | H.264 | object detection (single-stream input, -an, big probesize) |
| .21 | cam21 | ANNKE C500 5MP | H.264 | placement TBD |
| .22 | cam22 | ANNKE C500 5MP | H.264 | object detection + face recognition (server room). NTP synced 2026-05-10. |
| .23, .24 | TBD | ANNKE C500 5MP | H.264 | spare |

cam17 + cam22 detect at 1920x1080. Other cams 640x480 sub.

Frigate native face_recognition: DISABLED globally + per-cam (Double Take owns). ArcFace 261MB stripped.

## 4. Face Recognition Stack — Tuned 2026-05-10

Path B: Double Take + CompreFace. CPU-only on k3frigate.

### Pipeline

```
[cam17/cam22 person] -> Frigate event -> mosquitto.frigate -> Double Take
                                                                 |
                                                                 v
                                                       fetches snapshot.jpg?crop=1
                                                                 |
                                                                 v
                                                       POST cropped person to CompreFace
                                                                 |
                                                                 v
                                                       match("Name" >=60%) or unknown
                                                                 |
                                                                 v
                                                       writes Frigate event sub_label
```

### Components

| Namespace | Component | Image | Memory limit |
|-----------|-----------|-------|--------------|
| frigate | mosquitto | eclipse-mosquitto:2 | 128Mi |
| frigate | double-take | skrashevich/double-take:latest | 512Mi |
| face-recognition | postgres | postgres:16-alpine | 512Mi |
| face-recognition | compreface-core | exadel/compreface-core:1.2.0 (CPU build) | 4Gi |
| face-recognition | compreface-admin | exadel/compreface-admin:1.2.0 | 512Mi |
| face-recognition | compreface-api | exadel/compreface-api:1.2.0 | 512Mi |
| face-recognition | compreface-fe | exadel/compreface-fe:1.2.0 | 128Mi |

### DT config tuning (2026-05-10)

- `attempts.latest: 0` (was 8) — skip whole-frame polling, use ONLY cropped event snapshots
- `attempts.snapshot: 12` (was 5)
- cam11/12/13/15/16/19_yicam masked — DT skips non-face cams
- `face_plugins:` empty (was gender,age,mask) — 2-3x faster
- CompreFace timeout 30s (was 15s)
- `det_prob_threshold: 0.6`, `match.confidence: 60`, `unknown.confidence: 30`, `min_area: 5000`

### Face library hygiene — purge + retrain (2026-05-10)

Pre-tune captures used whole-frame images (no `crop=1`) — embeddings represented whole-room visuals not faces.

Cleanup:
- 9 Ivalin faces deleted via `DELETE /api/v1/recognition/faces?subject=Ivalin`
- DT `/mnt/frigate/double-take/{latest,matches,train}/*` wiped
- DT restarted

Retrain: walk past cam17/cam22 frontal → DT captures cropped person → label via DT UI → 5-10 cropped samples per family member. Or upload phone selfies via CompreFace UI.

### Storage

- Postgres data: hostPath `/mnt/frigate/compreface/postgres`
- Postgres backups: hostPath `/mnt/frigate/compreface/postgres-backups` (CronJob daily 3am, gzip, 14-day)
- DT config + matches: hostPath `/mnt/frigate/double-take`

### UI access

- LAN: http://compreface.lan + http://double-take.lan via Traefik IngressRoute. Requires `/etc/hosts` entry `192.168.100.200 compreface.lan double-take.lan`.
- External (Cloudflare Tunnel) — pending.

### Where matches show in Frigate UI

- Review event card: `person` → `person · Name`
- Explore filter: Sub Labels dropdown
- Event detail drawer: sub_label badge
- MQTT topic `double-take/matches` for automation

## 5. Dedicated Frigate MQTT broker — by design

Local mosquitto in `frigate` ns. Originally workaround for split-brain (now fixed) but kept by design:
- Low-latency video event processing (sub-ms loopback vs cross-node VXLAN)
- Isolation: Frigate independent of phone-hosted hydroflow broker
- Blast-radius separation

Two brokers, two domains:
- `mosquitto.frigate` — Frigate, Double Take
- `mosquitto.hydroflow` — ESP32 sensors, hydroflow-backend, chickenflow-backend

## 6. HydroFlow Postgres Migration (2026-05-10)

Migrated from one62 (phone eMMC, 5Gi) to k3frigate (SATA, 10Gi `/mnt/frigate/hydroflow-postgres`, mem 1Gi).

Process: deploy new pod with label `instance=new` on k3frigate -> pg_dump|psql in-cluster -> verify 7 tables row-counts match -> patch Service selector -> restart hydroflow-backend + chickenflow-backend -> scale old to 0 (kept 48h rollback).

Backup CronJob `hydroflow-postgres-backup` (hydroflow ns):
- 03:30 daily
- pg_dump | gzip | scp to `user@192.168.100.62:/home/user/backups/hydroflow-postgres/`
- 14-day retention (remote `find -mtime`)
- Reuses `jumphost-ssh-key` secret

Future: when SP4 onboarded as 3rd CP (k3node2), migrate postgres to SP4 = proper platform-service home.

## 7. Production Polish (2026-05-09 to 2026-05-10)

- Liveness probes (TCP-socket) on all 7 face-rec deployments
- k3master disk pruned 82% -> 73% (8 GB containerd cache freed)
- Frigate /dev/shm 256Mi -> 512Mi
- Stale errored pods + 3 failed Jobs cleaned
- Node labels: homelab/role={control-plane, video, mqtt, db, app}
- guitar-backend Service LoadBalancer -> ClusterIP (was stuck pending — .202 collision)
- Cam17 codec comment updated
- frigate-backup CronJob target IP corrected .152 -> .62

## 8. Known Open Issues

- k8s version skew (k3master/k3frigate 1.34.3, phones 1.35.0). Upgrade CP separate session.
- `default` namespace grab-bag (gitea, loki, uptime-kuma, open-webui, guitar). Migration heavy.
- Single CP SPOF — Phase 2 HA blocked on SP4.
- Multi-arch image discipline — process/CI rule pending.
- GardenFlow not deployed — tutor-track milestone.
- Locally-administered MAC clusters at .41/.51/.133/.140/.179.
- cam15 chronic RTSP outage — physical check needed.
- Cloudflare Tunnel routes for compreface/double-take external access — pending.
- HydroFlow postgres weak pwd `Ch@ngeMe!2025` — rotation overdue.

## 9. Credentials (encrypted in ~/homelab-docs/credentials.md.age)

Pending re-encrypt 2026-05-10:
- 3x ZyXEL switch admin pwd + per-switch SNMP Get community
- CompreFace postgres: compreface / eKtCcoTi63leS1d0Jwh5JQI2 / db frs
- CompreFace API key (frigate-faces): af6e2d86-3bba-434a-a7d4-d6c6b4ee19d5
- CompreFace UI superadmin (user-set during bootstrap)
- HydroFlow postgres: hydroflow_user / Ch@ngeMe!2025 / db hydroflow (rotation overdue)
- Double Take UI: SECURE=false (no auth) — tighten before external exposure
