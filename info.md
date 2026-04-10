# k3master — Machine Reference

Last updated: 2026-04-09

Machine-level reference for the k3master control plane node.
For cluster topology, services, and deployments see the k8s-ops skill (`~/.claude/skills/k8s-ops/`).

---

## Hardware

| Component | Details |
|-----------|---------|
| Model | Lenovo 81X3 (IdeaPad) — headless laptop |
| CPU | Intel Core i5-1035G1 @ 1.00GHz (4 cores / 8 threads, turbo 3.6GHz) |
| RAM | 7.4 GB DDR4 |
| Storage | 238.5 GB NVMe SSD (LVM: 100 GB root) |
| GPU | Intel Iris Plus G1 (Ice Lake, integrated) — QSV for Frigate video decode |
| Battery | BAT0 — Full (100%), always plugged in |
| Network | wan0 (Ethernet, 192.168.100.52/24), wlp0s20f3 (Wi-Fi, DOWN) |
| USB | 3x phone tethering (usb-one6t 10.0.1.1/30, usb-one62 10.0.2.1/30, usb-one61 10.0.3.1/30) |
| External HDD | 1.8 TB mounted at /mnt/frigate (Frigate recordings + config) |

---

## Operating System

| Field | Value |
|-------|-------|
| OS | Ubuntu 24.04.3 LTS (Noble Numbat) |
| Kernel | 6.8.0-101-generic (x86_64, PREEMPT_DYNAMIC) |
| Hostname | k3master |
| Shell | bash |
| DNS | systemd-resolved (127.0.0.53) |
| Swap | 4 GB |

---

## Installed Software

| Package | Version | Notes |
|---------|---------|-------|
| K3s | v1.34.3+k3s3 | Lightweight Kubernetes |
| Docker | 28.2.2 | Container engine + buildx |
| BuildKit | v0.27.1 | Multi-arch builder backend |
| Buildx | 0.31.1 | Docker build plugin |
| containerd | 1.7.28 / 2.1.5-k3s1 | System + K3s runtimes |
| Helm | v3.20.0 | K8s package manager |
| kubectl | v1.34.3+k3s3 | K8s CLI |
| Git | 2.43.0 | Version control |
| Python | 3.12.3 | Scripting, pip, pipx |
| tmux | — | Terminal multiplexer (gruvbox theme) |
| pmbootstrap | — | postmarketOS build tool (phone images) |
| OpenSSH | — | Server running, key-based auth |
| Claude Code CLI | — | AI pair programming |

---

## Systemd Services

| Service | State |
|---------|-------|
| k3s.service | active (K3s server) |
| docker.service | active |
| containerd.service | active |
| ssh.service | active |

---

## Cron Jobs

```
0 3 * * *  ~/backup-to-jumpbox.sh >> ~/backup.log 2>&1
```
Daily 3 AM: exports cluster state YAMLs + homelab-config to jumphost (192.168.100.152:~/backups/).

---

## Automation Scripts

| Script | Purpose |
|--------|---------|
| `~/backup-to-jumpbox.sh` | Nightly cluster state backup to jumphost |
| `~/redeploy-all.sh` | Full guitar-app rebuild (tar+SCP workflow) |
| `~/redeploy-backend.sh` | Guitar backend only rebuild |
| `~/redeploy-frontend.sh` | Guitar frontend only rebuild |

---

## SSH Config (~/.ssh/config)

```
Host one6t  → 10.0.1.2 (user, ed25519 key)
Host one62  → 10.0.2.2 (user, ed25519 key)
Host one61  → 10.0.3.2 (user, ed25519 key)
```

---

## Docker Configuration

| File | Contents |
|------|----------|
| `/etc/docker/daemon.json` | `{"insecure-registries": ["192.168.100.206"]}` |
| `~/buildkitd.toml` | Buildx builder config with insecure registry entries |
| `~/.docker/config.json` | Docker auth credentials |

Buildx builder: `hydroflow-builder` (docker-container driver, all platforms).
