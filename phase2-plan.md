# Phase 2 Plan — k3s HA Control Plane + Frigate Onboarding

Last updated: 2026-05-12 (HA migration complete)

**Source of truth:** Drive *Master Network & Cluster Architecture Document V14* (id `1JV54t8W2TzOq1Vg7Vubjk8plC4gyflJGttkPwV-qkrE`). Local snapshot: `v14-snapshot.md`.

**Operational runbook (checklists, bootstrap order, prep steps):** `~/.claude/skills/k8s-ops/references/phase2-migration.md`. This file holds *decisions, open blockers, validation gates* only.

**Archival trigger:** once validation criteria all green, archive this file under `homelab-docs/archive/`.

---

## Status snapshot 2026-04-29

| Workstream | Status |
|---|---|
| **Frigate onboarding to GPU node** | ✅ Done. Pod runs on `k3frigate` (i5-6600 + GTX 1050Ti) with ONNX/TensorRT detector (yolov9s @ 320, ~16 ms inference). 24 h soak started 2026-04-29T05:26Z. |
| **k3s split-brain fix on k3master** | ✅ Done. `node-ip` flipped 10.0.1.1 → 192.168.100.52, `flannel-iface` flipped `usb-one6t` → `wan0`. Cross-node VXLAN now on gigabit LAN. SOP recorded in `network.md`. |
| **3-node HA control plane** | ✅ **DONE 2026-05-12.** k3master + k3frigate + k3SP4 all control-plane,etcd. Phones (one6t/one61/one62) rejoined as agents via USB peer IPs (10.0.X.1, --flannel-iface usb0). 6 nodes Ready. Datastore = embedded etcd, 3-member quorum. Validation: `kubectl get nodes` shows 3 CP + 3 agents Ready. See cluster-improvements row 58 for full reroll log. |
| **Phone tethers as cluster workers** | ⚠ Degraded. Kubelets still reach apiserver via tether (still answers locally), but cross-node Pod-to-Pod traffic from k3master/k3frigate to phone Pod IPs broken since the .52 flip. Pods (mosquitto, postgres, gitea, hydroflow-backend) keep running locally. Re-onboard / static-route fix is a follow-up project. |

## Goal

Cluster from **single-master sqlite** → **3-node embedded etcd HA control plane** + Frigate NVR with GPU acceleration on the i5-6600 node.

**Node naming reality:** V14 plan called the i5-6600 host `k3node1`; actual deployed hostname is `k3frigate`. Treat the two as synonymous in this doc — `k3frigate` is canonical going forward.

---

## Decisions locked

| # | Decision | Value |
|---|----------|-------|
| 1 | Phones role | OnePlus 6 USB-tethered to k3master = WAN failover only (RNDIS subnet, NAT). No longer cluster workers. |
| 2 | HA control plane | k3node1 + k3node2 join as control-plane peers → 3-node embedded etcd quorum |
| 3 | OS for new nodes | Ubuntu Server 24.04 LTS |
| 4 | Workload migration | Frigate moves to k3node1 for 1050Ti TensorRT acceleration |
| 5 | i5-6600 status | Ubuntu LTS already installed and ready |
| 6 | Cameras | Flat `192.168.100.0/24` via Service Room Zyxel. VLAN 30 deferred. |
| 7 | Datastore | Embedded etcd (M1). M2 Postgres rejected — extra SPOF. M3 single-master rejected — no HA. |
| 8 | SP4 location | Inverter switch location (former ZyXEL #3 spot). Doubles as wall-mount kiosk. |
| 9 | SP4 networking | Original Surface Dock LAN (gigabit). Wi-Fi off for cluster traffic. |

---

## Hardware inventory

### k3master (.52, IdeaPad Flex 5)
- Functional master, serving live apps (workload inventory not yet captured)
- USB phone tether on RNDIS subnet, `ip_forward` enabled
- Datastore: default sqlite — must change for HA
- wan0 = UE306, MAC `9c:69:d3:4a:f4:a9`

### k3frigate (.56, i5-6600 + GTX 1050Ti) — formerly named k3node1 in V14
- Ubuntu 24.04.4 LTS, kernel 6.8.0-110-generic
- GPU: NVIDIA 1050Ti (Pascal). Driver 535.288.01, CUDA 12.2 runtime
- nvidia-container-toolkit + `runtimeClassName: nvidia` in use; nvidia-device-plugin DS active
- Storage: 2TB internal SATA mounted `/mnt/frigate` (recordings + config + sqlite DB + model_cache)
- Joined cluster as **agent** 2026-04-28; not (yet) a control-plane peer
- Frigate pod live since 2026-04-29 ~04:30Z (ONNX yolov9s 320, ~16 ms inference, ~880 MiB GPU mem)

### k3node2 (Surface Pro 4, 192.168.100.53)
- 8GB RAM / 256GB SSD
- **Ubuntu Server 24.04 LTS installed** (Fedora wiped). User `ivalin`. Confirmed 2026-05-12.
- Surface Dock LAN plugged in (gigabit). High ping RTT (~1.2s) on first probe → suspect NIC power-save / no sleep mask yet — to be fixed in pre-bootstrap pass.
- sshd not yet enabled — user opens manually 2026-05-12.
- Hostname pending rename to `k3node2` (currently default Ubuntu install hostname).
- Optional kiosk dashboard on touchscreen — see B3.
- **Concern:** dual-role budget tight (~3GB headroom)

---

## Open blockers

### B1. Datastore migration

V14 locks **M1 embedded etcd**. k3master `--cluster-init`; k3node1 + k3node2 join as `--server`. Outage during k3master reinit; backup gates everything. Procedure in skill runbook.

### B2. Inverter location networking

ZyXEL #3 removed in V12. Options:
- (a) Re-deploy ZyXEL #3 + cable run from ZyXEL #1
- (b) Single cable from ZyXEL #1 to SP4 dock direct (sufficient if only SP4 there)

### B3. SP4 dual-role

- (a) Combined: control plane + kiosk on same SP4 (8GB tight, browser noisy)
- (b) Split: SP4 control plane only, separate cheap display (old phone?) for kiosk

### B4. Frigate storage target (V14 specifies internal 4TB SATA on k3node1)

Verify SMART status pre-deploy. Backup target = 4TB USB 2.0 HDD on NAS Jumphost.

### B5. k3master backup state

Verify nightly cron output before reinit. Backup steps in skill runbook.

### B6. Workload inventory

Enumerate before migration (commands in skill runbook).

---

## Phase 1 outstanding

| # | Task | Status |
|---|------|--------|
| 1 | BE6500 model RN02 / FW 1.0.43 | Done |
| 2 | Backup taken (xmir-patcher option 4) | Done |
| 3 | SSH unlock (non-persistent — re-run after reboot) | Done |
| 4 | Auto-update kill (`INITTED_OTA=1`, cron stripped, DNS sinkhole) | Done |
| 7 | DHCP reservations for `.52`/`.56` | Done |
| 8 | WAN + LAN port label verify | **Pending** |
| 9 | VLAN 30 cameras | Deferred to Phase 2+ |

---

## Validation criteria — Phase 2 done

- All 3 nodes `Ready,control-plane` ⏳ (only k3master is control-plane today; k3frigate is agent; k3node2 not provisioned)
- Single-node loss survivable (test by stopping k3s on one) ⏳
- All pre-migration k3master workloads Running post-cutover (zero data loss) ✅ (Frigate cutover 2026-04-29; phone-hosted pods still Running locally)
- Frigate detecting on live cameras via TensorRT (no CPU fallback) ✅ (ONNX detector with `TensorrtExecutionProvider` available, ~16 ms inference; soak ends 2026-04-30T05:26Z)
- Frigate recordings persisting on internal SATA (V14 said 4TB; actual = 2TB on k3frigate) ✅
- SP4 kiosk live (if dual-role chosen) ⏳
- Phase 1 Task 8 (port label verify) closed ⏳
- BE6500 DHCP reservation for k3node2 added ⏳
- Drive doc V15 updated by Gemini reflecting executed state ⏳

## Follow-up work (added 2026-04-29)

- **P7 — decommission 1.8TB external HDD on k3master** once Frigate soak passes (was the previous recordings target).
- **Phone-tether re-onboarding or static-route fix.** Phones now in degraded cross-node state because k3master flipped to LAN node-ip. Either add `10.0.X.0/30 via 192.168.100.52` static routes on k3frigate (and configure forwarding on k3master) or re-onboard the phones via a different network path.
- **HA control plane (M1 embedded etcd).** Still on the original V14 plan; awaits k3master reinit + k3node2 (SP4) provisioning.

---

## Risks

- **k3master reinit downtime** — backup correctness gates whole migration
- **SP4 8GB RAM** — dual-role close to budget; watch OOM
- **Surface Dock + Linux** — verify LAN port on Ubuntu before commit
- **Lid-close thermal** — stand laptop open if heat issue
- **Frigate disk writes** — SMART monitor on 4TB SATA before 24/7 ops
- **BE6500 SSH unlock non-persistent** — re-run xmir-patcher Option 2 after router reboot
- **WAN failover SPOF** — OP6 USB tether routes via k3master; if k3master down during primary WAN outage, no failover

---

## Errata vs. Drive briefing

- BE6500 router MAC: live ARP `44:f7:70:b9:c8:bc`. Briefing claimed `c8:4d:44:35:0b:46` (old WAN dongle, still in udev for rollback) — corrected here.
- GardenFlow / `/home/ivalin/Documents/` references in briefing do not apply.
