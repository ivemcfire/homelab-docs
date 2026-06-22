# Master Network & Cluster Architecture Document V17.5

Status: Phase 3. Service room switch live; Attic + Inverter benched. Face Recognition Stack live + tuned. Cross-namespace MQTT split-brain fixed via static routes. HydroFlow postgres on k3frigate. **Control-plane k3s upgraded to v1.35.5+k3s1 (etcd-leak fix). SP4 (.53) RETIRED — Phase 2 HA path dead. Phone nodes tainted to keep general workloads off the 2A-limited phones. k3master LAN migrated to isolated RTL8156 2.5G NIC.**

Last updated: 2026-06-22. Supersedes V17.4 (dated 2026-05-10).

> **Changes since V17.4** (see §10 for detail):
> - k3s control plane upgraded 1.34.3 → **v1.35.5+k3s1** (leaked-etcd-backend CPU bug fix); phones on 1.35.0+k3s3.
> - **SP4 / k3node2 retired 2026-06-08** (swollen battery, digitizer dead). The "SP4 becomes 3rd control-plane / postgres home" plan is cancelled; single-CP SPOF is now permanent until a replacement node is sourced.
> - **Phone nodes tainted** `node-role/phone=true:NoSchedule` (2026-06-22) — one62 was browning out under load on its 2A supply. Only arm64-native workloads + DaemonSets tolerate the taint; general cluster services pinned to amd64 nodes.
> - **k3master LAN NIC migrated** (2026-06-09) to RTL8156 2.5G on an isolated PCI controller, off the phone-tether controller that was wedging.
> - §9 credentials: inlined plaintext secrets removed — values live only in `credentials.md.age`.

## 1. Physical Infrastructure & Network Core

- Gateway: Xiaomi BE6500 (192.168.100.1). Locked. OTA cron stripped, DNS sinkhole, INITTED_OTA=1. xmir-patcher SSH unlock only — NOT full OpenWrt; mesh kept; no full firewall ruleset.

### Switching (ZyXEL GS1200-8 v3)

| Hostname | Mgmt IP | MAC | Status |
|----------|---------|-----|--------|
| zyxel-service | 192.168.100.2 | c0:2e:5f:57:fe:d4 | LIVE service room (8/8 ports used) |
| zyxel-attic | 192.168.100.3 | c0:2e:5f:58:01:7f | Benched, placement pending |
| zyxel-inverter | 192.168.100.4 | c0:2e:5f:58:01:79 | Benched, placement pending |

All 3 hardened: HTTPS only, HTTP/SNMPv2c-write/Telnet off, EEE off, Loop Guard + IGMP Snooping on, SNMPv2c read with per-switch random Get community.

### k3master LAN NIC — migrated 2026-06-09

- LAN moved to a **new RTL8156 2.5G USB adapter on an ISOLATED USB controller** (`0000:00:0d.0`), deliberately OFF the phone-tether controller (`0000:00:14.0`) that was wedging and only recovered via PCI reset.
- `wan0` defined by **netplan match-by-MAC**; the link watchdog was retargeted to the new interface.
- **Open:** link negotiated at 1G, not 2.5G (suspect cable). Watch for `-71` (ENOLINK) recurrence — prior root cause of the cascade.

## 2. k3s Cluster Topology

| Hostname | Hardware | IP | Role | k3s |
|----------|----------|-----|------|-----|
| k3master | IdeaPad Flex 5 (i5-1035G1) | 192.168.100.52 | Server / Control plane / Chrony NTP for cams | v1.35.5+k3s1 |
| k3frigate | i5-6600 + GTX 1050Ti | 192.168.100.56 | Agent — Frigate compute + face-recognition + hydroflow postgres | v1.35.5+k3s1 |
| one61 | OnePlus 6, sdm845, 8GB | 10.0.3.2 (tether) | Agent (arm64) — Adreno inference / arm64 builds | v1.35.0+k3s3 |
| one62 | OnePlus 6, sdm845, 8GB | 10.0.2.2 (tether) | Agent (arm64) — **power-fault, see §8** | v1.35.0+k3s3 |
| one6t | OnePlus 6T, sdm845, 6GB | 10.0.1.2 (tether) | Agent (arm64) — mqtt-inference-worker, hydroflow-backend, mosquitto, gitea | v1.35.0+k3s3 |
| ~~k3sp4~~ | ~~Surface Pro 4~~ | ~~192.168.100.53~~ | **RETIRED 2026-06-08** (swollen battery / dead digitizer). Phantom etcd member — `kubectl delete node k3sp4`. | — |
| pc-windows | Win10, i5-10400F + GTX 1050Ti | 192.168.100.50 | Local-AI project scrapped 2026-05-11. Idle / not in cluster. | — |
| Jumphost | (admin host) | 192.168.100.62 | Network management + backup target; redundant Tailscale subnet router | — |
| acergo16 | Fedora laptop | 192.168.100.65 | Admin station | — |

### Phone node scheduling discipline — taint added 2026-06-22

All 3 phones are tainted `node-role/phone=true:NoSchedule`. Rationale: one62 repeatedly browns out because its workload draws more than the **2A supply** can deliver. The scheduler reads phones as idle (low CPU requests) and piles general services onto them; those spike power and trip the node, flooding alerts.

- **Tolerate the taint (stay on phones):** arm64-locked deploys `gpu-inference/mqtt-inference-worker`, `buildkit/builder-arm64`; DaemonSets `metallb-system/speaker`, `smarter-device-manager`, `monitoring/node-exporter` (node-exporter already tolerated all NoSchedule).
- **Evicted to amd64 (k3master/k3frigate):** traefik, metrics-server, prometheus-operator, loki-chunks-cache, loki-results-cache, local-path-provisioner, metallb controller, reloader.
- **To run anything new on a phone:** set both `nodeSelector kubernetes.io/arch: arm64` (if arm-only) AND the toleration `{key: node-role/phone, operator: Exists, effect: NoSchedule}`.
- Caveat: `speaker`/`node-exporter` are Helm-managed; toleration patches may revert on chart upgrade.

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

L2 advertisement on `wan0` (k3master). Pool `homelab-pool` 192.168.100.200-220. Memberlist key rotated 2026-05-10.

## 3. Frigate NVR + Camera Inventory

Software: ghcr.io/blakeblackshear/frigate:0.17.1-tensorrt
Compute: k3frigate (1050Ti, 4GB VRAM)
Mem limit: 4Gi. /dev/shm: 512Mi.
NTP server for cams: k3master 192.168.100.52.
Hardening per cam: null gateway, null DNS, NTP=192.168.100.52, P2P/Cloud disabled.
Config source of truth: hostPath `/mnt/frigate/config/config.yml` on k3frigate (ConfigMap can lag — read/edit the hostPath).

### Camera IP allocation (192.168.100.11-29)

| IP | Cam | Model | Codec | Role |
|----|-----|-------|-------|------|
| .11-.13, .15, .16 | cam11-16 (XMeye) | NETSurveillance | H.264 | object detection only |
| .17 | cam17_reolink | Reolink 4K | H.265 | object detection + face recognition |
| .19 | cam19_yicam | Xiaomi Yi (yi-hack) | H.264 | object detection (single-stream input, -an, big probesize) |
| .21 | cam21 | ANNKE C500 5MP | H.264 | placement TBD |
| .22 | cam22 | ANNKE C500 5MP | H.264 | object detection + face recognition (server room) |
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

Local mosquitto in `frigate` ns. Kept by design even after split-brain fix:
- Low-latency video event processing (sub-ms loopback vs cross-node VXLAN)
- Isolation: Frigate independent of phone-hosted hydroflow broker
- Blast-radius separation

Two brokers, two domains:
- `mosquitto.frigate` — Frigate, Double Take
- `mosquitto.hydroflow` — ESP32 sensors, hydroflow-backend, chickenflow-backend

## 6. HydroFlow Postgres (migrated 2026-05-10)

On k3frigate (SATA, 10Gi `/mnt/frigate/hydroflow-postgres`, mem 1Gi). Migrated off one62 phone eMMC.

Backup CronJob `hydroflow-postgres-backup` (hydroflow ns):
- 03:30 daily
- pg_dump | gzip | scp to `user@192.168.100.62:/home/user/backups/hydroflow-postgres/`
- 14-day retention; reuses `jumphost-ssh-key` secret

> **NOTE (V17.5):** V17.4 planned migrating this postgres to SP4 once it became the 3rd control plane. **SP4 is retired** — that plan is void. Postgres stays on k3frigate until a replacement platform node exists.

## 7. Monitoring & Alerting

- In-cluster Grafana `monitor.ayurforlife.eu` (LB .201, ds uid `prometheus`); standalone Grafana on jumphost:3000 pinned 11.5.2 (G12 saturates .62).
- Alertmanager is a hand-managed Deployment; config in cm `monitoring/alertmanager-config`. Edit CM + rollout restart.
- Prometheus scrape intervals backed off to 5m (was 10s spiking k3s-server CPU).
- **Email notifications DISABLED 2026-06-22 (temporary)** — flood of real criticals (CameraDown, one62 NodeNotReady) + KubeJobFailed. `critical` and infra-down routes flipped to `blackhole`; backup at `k3master:/tmp/am-backup-20260622.yaml`. Re-enable: set those routes back to `receiver: email` + rollout restart.
- frigate-backup: baked-in rsync image, dest .52; cloudflared 2 replicas + maxSurge 0; KubeJobFailed emails.

## 8. Known Open Issues

- **Single CP SPOF — now permanent-ish.** SP4 (k3node2) was the planned 3rd control plane; it's retired. HA needs a new node. Datastore is embedded etcd on a single server.
- **one62 power fault.** Browns out / shuts off under load on its 2A supply; NotReady for stretches (was down 2026-06-15 → 22). Software mitigated via the phone taint (§2); real fix is a beefier PSU. Powered USB hub PD is capped ~70%.
- Minor k8s version skew (servers v1.35.5+k3s1, phones v1.35.0+k3s3) — phones lag intentionally; upgrade when convenient.
- `default` namespace grab-bag (gitea, loki, uptime-kuma, open-webui, guitar). Migration heavy.
- Multi-arch image discipline — process/CI rule pending.
- GardenFlow not deployed — tutor-track milestone.
- cam15 chronic RTSP outage — physical check needed.
- Cloudflare Tunnel routes for compreface/double-take external access — pending.
- **Credential rotation overdue** — HydroFlow postgres weak pwd; CompreFace creds; ZyXEL admin/SNMP. See §9.
- Double Take UI: SECURE=false (no auth) — tighten before external exposure.
- k3master LAN linked at 1G not 2.5G (cable) — see §1.

## 9. Credentials

Stored encrypted in `~/homelab-docs/credentials.md.age`. **Plaintext values are NOT inlined in this doc (changed in V17.5).** Decrypt the `.age` file for:
- 3× ZyXEL switch admin pwd + per-switch SNMP Get community
- CompreFace postgres (db `frs`) + CompreFace API key (frigate-faces) + UI superadmin
- HydroFlow postgres (db `hydroflow`) — **rotation overdue, weak password**
- Double Take UI (currently no auth)

Action: rotate HydroFlow postgres + CompreFace creds, then re-encrypt `credentials.md.age`.

## 10. Changelog V17.4 → V17.5 (2026-06-22)

- **k3s upgrade:** control plane 1.34.3 → v1.35.5+k3s1 (fixes leaked-etcd-backend 100% CPU bug). Phones remain v1.35.0+k3s3.
- **SP4 retired 2026-06-08:** removed as future 3rd CP / postgres host; SPOF documented as open issue. Phantom etcd member to clean (`kubectl delete node k3sp4`).
- **Phone taint 2026-06-22:** `node-role/phone=true:NoSchedule` on one61/62/6t; tolerations + workload re-pinning documented in §2.
- **k3master NIC migration 2026-06-09:** LAN on isolated RTL8156 2.5G controller (§1).
- **Alertmanager email disabled 2026-06-22** (§7).
- **§9 hardening:** inlined plaintext secrets removed; doc points to `credentials.md.age` only.
