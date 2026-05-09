# Master Network & Cluster Architecture Document V17.3

Status: Phase 3 in progress (Switch Deployment + Camera Hardening + Face Recognition Stack). Service room switch live; Attic and Inverter units benched. ANNKE C500 cam22 added at server room. Double Take + CompreFace face-recognition stack deployed on k3frigate.

Last updated: 2026-05-09. Supersedes V17.2 (cam22 attic typo corrected; face-recognition stack added; mosquitto cross-namespace MQTT issue resolved).

## 1. Physical Infrastructure & Network Core

- Gateway: Xiaomi BE6500 (192.168.100.1)
- Security state: Locked. OTA cron stripped, DNS sinkhole applied, persistent INITTED_OTA=1. xmir-patcher SSH unlock only — NOT full OpenWrt; mesh kept; no full firewall ruleset (Phase 2 VLAN segregation deferred).

### Switching Infrastructure (ZyXEL GS1200-8 v3)

| Hostname | Mgmt IP | MAC | Status |
|----------|---------|-----|--------|
| zyxel-service | 192.168.100.2 | c0:2e:5f:57:fe:d4 | LIVE service room (8/8 ports used) |
| zyxel-attic | 192.168.100.3 | c0:2e:5f:58:01:7f | Benched, placement pending |
| zyxel-inverter | 192.168.100.4 | c0:2e:5f:58:01:79 | Benched, placement pending |

Configuration (all 3): IGMP Snooping enabled, Loop Prevention enabled, password hardened (16-char alphanum), HTTPS only, HTTP/SNMPv2c-write/Telnet disabled, EEE disabled, Mgmt VID 1 (Phase 2 may change), SNMPv2c read with per-switch random Get community.

## 2. k3s Cluster Topology

| Hostname | Hardware | IP | Role |
|----------|----------|-----|------|
| k3master | IdeaPad Flex 5 (i5-1035G1) | 192.168.100.52 | Server / Control plane / Chrony NTP for cams |
| k3frigate | i5-6600 + GTX 1050Ti | 192.168.100.56 | Agent — Frigate compute + face-recognition stack |
| LocalAI | Win10, i5-10400F + RTX 3060 12GB | 192.168.100.50 | Local AI worker (Ollama) |
| Jumphost | (admin host) | 192.168.100.62 | Network management (was .152 — old binding still active, cleanup queued) |
| acergo16 | Fedora laptop | 192.168.100.65 | Admin station |

Phones (one61/62/6t at 10.0.x.2) USB-tethered to k3master, kubelet healthy but cross-node Pod traffic to phone Pod IPs broken since 2026-04-29 split-brain fix. Resolved at app level by deploying alternate brokers/services on amd64 nodes when the phone broker is unreachable from k3frigate (see §6).

## 3. Frigate NVR + Camera Inventory

Software: ghcr.io/blakeblackshear/frigate:0.17.1-tensorrt
Compute node: k3frigate (1050Ti, 4GB VRAM)
NTP server for cams: k3master 192.168.100.52 (Sofia/Bulgaria zone)
Hardening protocol per cam: null gateway, null DNS, NTP=192.168.100.52, P2P/Cloud disabled, ONVIF auto-discovery off if unused.

### Camera IP allocation (192.168.100.11–.29)

| IP | Cam | Model | Codec | Role |
|----|-----|-------|-------|------|
| .11–.13, .15, .16 | cam11–16 (XMeye) | NETSurveillance | H.264 | object detection |
| .17 | cam17_reolink | Reolink 4K | H.265 | object detection + face recognition |
| .19 | cam19_yicam | Xiaomi Yi (yi-hack) | H.264 | object detection |
| .21 | cam21 | ANNKE C500 5MP | H.264 | (placement TBD) |
| .22 | cam22 | ANNKE C500 5MP | H.264 | object detection + face recognition (server room) |
| .23, .24 | (placement TBD) | ANNKE C500 5MP | H.264 | spare |

Frigate detect resolution: 1920x1080 on cam17 + cam22 (face cams need detail). Other cams 640x480 sub stream.

### Frigate deployment polish (2026-05-09)

- Memory limit: 4Gi (was 5Gi during testing peak; steady ~3Gi)
- /dev/shm: 512Mi emptyDir (256Mi too small for 1080p detect)
- ArcFace 261MB model_cache stripped (Frigate native face_rec disabled — Double Take owns face)
- MQTT host: mosquitto.frigate.svc.cluster.local (was hydroflow's broker — unreachable from k3frigate)
- cam19_yicam config: single main stream input with `-an` audio drop + larger probesize (yi-hack codec quirk)

## 4. Face Recognition Stack (NEW in V17.3)

Path B per session decision: Double Take + CompreFace replace Frigate native face_recognition. CPU-only (1050Ti VRAM reserved for Frigate detection). All workloads pinned to k3frigate.

Pipeline:
```
[cam17/cam22 person] -> Frigate event -> mosquitto.frigate -> Double Take
                                                                |
                                                                v
                                                       fetches Frigate snapshot
                                                                |
                                                                v
                                                       POST /api/v1/recognition/recognize
                                                                |
                                                                v
                                                       CompreFace returns match("Name" 87%) or unknown
                                                                |
                                                                v
                                                       writes Frigate event sub_label
```

Components:

| Namespace | Component | Image | Memory limit |
|-----------|-----------|-------|--------------|
| frigate | mosquitto | eclipse-mosquitto:2 | 128Mi |
| frigate | double-take | skrashevich/double-take:latest | 512Mi |
| face-recognition | postgres | postgres:16-alpine | 512Mi |
| face-recognition | compreface-core | exadel/compreface-core:1.2.0 (CPU build) | 4Gi |
| face-recognition | compreface-admin | exadel/compreface-admin:1.2.0 | 512Mi |
| face-recognition | compreface-api | exadel/compreface-api:1.2.0 | 512Mi |
| face-recognition | compreface-fe | exadel/compreface-fe:1.2.0 | 128Mi |

Liveness probes: TCP-socket on all 7 deployments (initial delay 60–180s, period 30s, failure 3).

### Storage

- Postgres data: hostPath /mnt/frigate/compreface/postgres on k3frigate
- Postgres backups: hostPath /mnt/frigate/compreface/postgres-backups (CronJob daily 3am, gzip dump, 14-day retention)
- Double Take config + matches: hostPath /mnt/frigate/double-take on k3frigate

### UI access

- LAN: http://compreface.lan and http://double-take.lan (Traefik IngressRoute) — requires /etc/hosts entry `192.168.100.200 compreface.lan double-take.lan` on viewing device. NOT k3master .52.
- Bootstrap port-forward (k3master ssh): kubectl -n face-recognition port-forward --address 0.0.0.0 svc/compreface-fe 8001:80; kubectl -n frigate port-forward --address 0.0.0.0 svc/double-take 3000:3000.
- External (Cloudflare Tunnel) — NOT YET deployed for compreface/double-take. Pending decision.

### Day-to-day usage

| Task | UI |
|------|----|
| Watch live cams + recordings | Frigate (cams.ayurforlife.eu) |
| Filter clips by family member | Frigate UI search -> sub_label="Name" |
| Label new face captures | Double Take UI -> unknown thumbnails -> assign subject |
| Add/remove person | CompreFace UI -> Subjects tab |
| Add new face cam | Update Frigate config + Double Take frigate.cameras config |

## 5. Cross-namespace MQTT mitigation (k3frigate <-> phone broker)

Mosquitto in `hydroflow` ns runs on one6t (phone). After 2026-04-29 split-brain fix, k3frigate cannot reach one6t Pod IP via flannel VXLAN. Local mosquitto deployed in `frigate` namespace on k3frigate (eclipse-mosquitto:2, anonymous, no persistence). Frigate + Double Take repointed. HydroFlow consumers untouched (still use original broker).

## 6. Known issues + queued work

- MetalLB Memberlist UDP probes failing k3master <-> k3frigate (TCP works, gossip degraded). Investigate WireGuard mesh UDP rules.
- Jumphost dual-IP at .62 + .152 (legacy binding still answers ARP). Remove from netplan.
- Locally-administered MAC clusters at .41/.51/.133/.140/.179 — inventory before VLAN segmentation.
- cam22 NTP sync not applying via cam GUI hardening (timestamp drift visible). Re-set via raw IP since DNS nulled.
- cam15 chronic RTSP outage — cam offline (power/network), not Frigate side.
- Permanent ingress for face-rec UIs not yet routed via Cloudflare Tunnel.
- 18 GB /home/user on k3master — review/archive.
- ZyXEL #2 + #3 placement audit pending before applying V4 port maps to attic + inverter spots.

## 7. Credentials (encrypted in ~/homelab-docs/credentials.md.age)

Pending re-encrypt — added to working copy 2026-05-09:
- 3x ZyXEL switch admin pwd + per-switch SNMP Get community
- CompreFace postgres: compreface / eKtCcoTi63leS1d0Jwh5JQI2 / db frs
- CompreFace API key (frigate-faces app): af6e2d86-3bba-434a-a7d4-d6c6b4ee19d5
- CompreFace UI superadmin (user-set during bootstrap)
- Double Take UI: SECURE=false (no auth) — tighten before any external exposure
