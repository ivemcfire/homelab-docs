# Incident + Runbook — .52 USB/Power Cascade (multi-node reboot)

**Date:** 2026-06-07
**Symptom reported:** .52, .53, .56 down then rebooted; .53 + one61 still down; "overheating" suspected.
**Verdict:** NOT overheating. Root cause = **.52 USB/power fragility**, with several independent faults stacked on top (classic cascading-faults).

---

## Root cause

.52's **entire network rides USB on one xHCI controller**: the LAN NIC (`RTL8153`/`r8152`, udev name `wan0`) and **all three phone-node tethers** hang off controller `0000:00:14.0`. The second controller `0000:00:0d.0` (USB-C/Thunderbolt, 10 Gbps) sits **empty**. When any device on `00:14.0` errors hard, the xhci reset/recovery cascades to everything on it → LAN + all phone nodes drop together → `r8152` NETDEV WATCHDOG → reboot.

**Amplifier (confirmed):** .52 + .53 + the 3 phones were all powered from a **single 67W multiport USB charger** — oversubscribed (peak draw ~130W+). .52 is USB-C PD powered with a battery buffer (rides sags instead of hard-crashing). Kernel signature `usb 4-5-port3: Cannot enable. Maybe the USB cable is bad?` with **no over-current trip** = marginal power/signal quality under spikes, exactly what an undersized brick produces. Power sag turned small USB hiccups into full `err -110` cascades.

**Independent faults seen (don't attribute to one cause):**
- `r8152 wan0: Tx timeout → hub 4-5 err -110 → port3 cannot reset` (NIC-initiated collapse).
- **one6t** marginal USB cable: `device descriptor read/64, error -71` + `cdc_ncm transmit queue timed out` every ~2 min. Sibling **one62 on the same hub = 0 errors** → it's one6t's cable, not the hub/power.
- **.53** clean ordered poweroff (swollen battery), not a thermal trip.
- **.56** separate reboot 3h earlier (15:49), unrelated.
- **one61** down ~3 days — power-related (tied to this cascade); restored on repower.

**Fallout:** CoreDNS rescheduled onto phone **one62** (arm64, broken in-pod DNS) → cluster-wide DNS timeouts → `frigate/cloudflared` crashloop (`lookup ... on 10.43.0.10:53: i/o timeout`, exit 137). `kube-state-metrics` crashloop (1511×) was apiserver-unreachable (`dial 10.43.0.1:443: no route to host`) during .52 flapping — **not OOM**.

---

## Changes applied (software, durable)

| Fix | Where it lives | Notes |
|-----|----------------|-------|
| **CoreDNS off phones** — `replicas=2`, `nodeSelector arch=amd64` | `pin-coredns.{sh,service,timer}` on **.52 + .56** (`/usr/local/sbin/pin-coredns.sh`; timer OnBootSec=90s / every 10min) | k3s rewrites its bundled `coredns.yaml` on **every server start**, reverting manifest edits AND live patches → must re-pin via timer. |
| **USB auto-recovery watchdog** | `usb-net-watchdog.{sh,service}` on **.52** | Pings GW + .56; after ~60s LAN loss escalates L1 `r8152` rebind → L2 USB authorize cycle → L3 PCI remove/rescan of `0000:00:14.0`. `ENABLE_PCI_RESET=1`, `ENABLE_REBOOT_FALLBACK=0`. L1 tested live OK; L3 ran in prod and is survivable but can't fix a physically-gone NIC. |
| **Prometheus scrape → 5m** | Prometheus CR + 4 ServiceMonitors (kubelet **10s→5m**, node-exporter 15s→5m, frigate-exporter, fluent-bit) | The kubelet `/metrics/cadvisor` 10s scrape = `k3s-server` CPU surging 150-200% every 10s (heat, esp. SP4). Now flat ~104% baseline. **Helm-managed → reverts on chart upgrade.** |
| **iptsd masked on .53** | `systemctl mask iptsd@.service` | Stops `ipts_poll` CPU burn on the dead digitizer. (.53 now retired.) |
| **KSM counter reset** | — | Restarts were scar tissue from .52 flapping, not a live fault. |

## Changes applied (hardware/operator)

- **Power rebalanced:** phones off the .52 brick → now **.52 + phones only, 20-40W** into the 67W brick (was oversubscribed with .53 on it too).
- **.53 retired** (swollen battery). **etcd member LEFT in place** — repower to rejoin 3/3. BIOS had **no charge-limit option**; considering a **batteryless run** (may not POST / may brown-out under load — test before trusting as etcd).
- **one61** repowered → Ready. **.52** power-cycled (recovered from islanded NIC-dead state).

---

## Verification (end state)

- Nodes: k3master ✅, k3frigate ✅, one61/62/6t ✅; k3sp4 ⏸️ retired. `err -110 = 0` since reboot.
- CoreDNS: 2 replicas on master+frigate (amd64), durably pinned.
- `k3s-server`: flat ~104% on .52, no 10s surges.
- etcd quorum OK (2/3) — **but zero fault tolerance** while .53 is out (master+frigate both must stay up).

## Pending (physical)

1. **Move .52 NIC dock to the empty `0000:00:0d.0` (USB-C/TBT) controller** — isolates the gigabit NIC from the phone tethers so neither can wedge the other. The single highest-value fix.
2. **Reseat/replace one6t's USB-C cable** (the `error -71` flapper).
3. **.53 batteryless decision** — if it brown-outs under k3s load, not reliable as etcd; consider a higher-W USB-C→Surface-Connect PD adapter or keep retired.
4. **If .53 is gone long-term: add a 3rd amd64 etcd node** to restore HA fault tolerance.

## Runbook — if it recurs

- **3 nodes (incl phones) drop together + .52 unreachable = USB/power cascade, NOT heat.** Confirm: `sudo dmesg | grep -E 'err = -110|error -71|Tx timeout|Cannot enable'`.
- **.52 islanded** (kernel alive, NIC dead → invisible from LAN, no Tailscale either since it rides wan0): **physical power-cycle**. First thing on reboot: `journalctl -u usb-net-watchdog -b -1` to see what the watchdog tried.
- **Wedged-but-alive NIC**: watchdog auto-recovers (L1/L2/L3). Watch `journalctl -u usb-net-watchdog -f`.
- **CoreDNS on a phone again / DNS timeouts**: `systemctl start pin-coredns` (or wait ≤10min for the timer). Check `kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide`.
- **`k3s-server` 10s surges / SP4 overheating after a helm upgrade**: kubelet ServiceMonitor reverted to 10s — re-patch all 3 endpoints to `5m`.

_Cross-ref memories: homelab_usb_nics, feedback_phone_node_dns, monitoring_scrape_intervals, sp4_digitizer_damaged, feedback_cascading_faults._
