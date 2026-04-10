# Kubernetes Sidecar Pattern — Guitar Backend Observability

A real-world implementation of the Kubernetes sidecar pattern, built on a homelab k3s cluster running on repurposed OnePlus smartphones. This repository demonstrates structured log shipping, Prometheus metrics, and Loki integration — all wired together through a single additional container that travels with the application pod.

## The Problem This Solves

The guitar practice application (a FastAPI backend using librosa for audio processing) was experiencing intermittent OOMKill events caused by memory spikes during large file analysis. Diagnosing these crashes required manually checking `kubectl logs` and correlating timestamps with node metrics in Grafana — a slow, manual process with no alerting.

The goal was to build **automatic crash detection and log visibility** without modifying the application's core logic, and without deploying a heavyweight log agent on every node.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Guitar Backend Pod                          │
│                                                                     │
│   ┌──────────────────────┐          ┌───────────────────────────┐  │
│   │   guitar-backend     │  writes  │   fluent-bit-sidecar      │  │
│   │   (FastAPI + librosa)│ ───────► │   (log processor)         │  │
│   │                      │          │                           │  │
│   │   :8000 (HTTP API)   │          │   :2020 (FB health API)   │  │
│   │                      │          │   :2021 (Prometheus out)  │  │
│   └──────────────────────┘          └───────────────────────────┘  │
│             │                                    │                  │
│             └──────────────┬───────────────────-─┘                  │
│                            │                                        │
│                    shared-logs (emptyDir)                           │
│                    /var/log/guitar-backend/app.log                  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
           ┌─────────────────┼──────────────────┐
           ▼                 ▼                  ▼
      Loki :3100      Prometheus :2021      stdout
      (log store)     (metrics scrape)   (kubectl logs)
           │
           ▼
      Grafana :80
      (visualization + alerting)
```

The critical mechanism is the **shared emptyDir volume**. Both containers mount it at `/var/log/guitar-backend`. The application writes a structured JSON log file there; Fluent Bit tails that file and fans the data out to three outputs simultaneously.

## Why the Sidecar Pattern?

The alternative is a **DaemonSet log collector** — a single Fluentd or Fluent Bit pod running on each node and scraping every container's stdout. That approach works well for homogeneous workloads. The sidecar pattern is better here for three reasons.

First, **structured log files vs. unstructured stdout**. The node-level approach reads raw stdout, which means you have to parse unstructured text to extract fields like `severity`, `duration_ms`, or `file_size_mb`. With a sidecar, the application writes JSON directly to a file, and Fluent Bit reads it already structured — no regex parsing required.

Second, **portability**. The logging configuration is part of the pod spec. If you move the guitar-backend to a different node (or a different cluster entirely), the sidecar comes with it. No DaemonSet pre-installed on the destination node required.

Third, **separation of concerns**. The guitar-backend team (you) owns the entire observability stack for that one service. No shared Fluentd config to edit, no risk of your config change affecting another pod's log shipping.

## Cluster Context

This runs on a k3s cluster built from a Lenovo laptop (control plane) and three OnePlus smartphones running postmarketOS, connected via USB ethernet with routed /30 point-to-point links. MetalLB provides LoadBalancer IPs on the LAN. Full cluster build documented at [ivemcfire.github.io](https://ivemcfire.github.io).

| Node | Hardware | Role |
|------|----------|------|
| k3master | Lenovo i7-10750H, 8GB RAM | control-plane + workloads |
| one6t | OnePlus 6T, Snapdragon 845, 6GB | worker |
| one62 | OnePlus 6, Snapdragon 845, 8GB | worker |
| one61 | OnePlus 6, Snapdragon 845, 8GB | worker |

## Prerequisites

You need a running k3s cluster with MetalLB and Prometheus already deployed. Loki and Fluent Bit will be installed by the manifests in this repo.

If your guitar-backend currently only logs to stdout, you need to add a file handler before the sidecar can read structured logs. A minimal Python example:

```python
import logging, json, os

log_file = os.getenv("LOG_FILE", "/var/log/guitar-backend/app.log")

class JsonFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "time": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
        })

handler = logging.FileHandler(log_file)
handler.setFormatter(JsonFormatter())
logging.getLogger().addHandler(handler)
```

## Deployment

Apply the manifests in order. Each file is numbered and heavily commented to explain the reasoning behind each decision.

```bash
# 1. Create the Fluent Bit ConfigMap
sudo kubectl apply -f manifests/01-fluent-bit-config.yaml

# 2. Deploy Loki (log storage)
sudo kubectl apply -f manifests/02-loki.yaml

# Wait for Loki to be ready before deploying the sidecar that ships to it
sudo kubectl rollout status deployment/loki

# 3. Deploy guitar-backend with the Fluent Bit sidecar
sudo kubectl apply -f manifests/03-guitar-backend-with-sidecar.yaml

# 4. Wire up Prometheus scraping (choose ServiceMonitor or ConfigMap patch)
sudo kubectl apply -f manifests/04-prometheus-scrape.yaml
```

## Verification

After deployment, you should have a pod with two running containers. Confirm with:

```bash
# You should see 2/2 READY for the guitar-backend pod
sudo kubectl get pods -l app=guitar-backend

# Stream logs from each container separately
sudo kubectl logs -l app=guitar-backend -c guitar-backend --follow
sudo kubectl logs -l app=guitar-backend -c fluent-bit-sidecar --follow

# Check Fluent Bit's health API (runs on the same pod IP, port 2020)
POD_IP=$(sudo kubectl get pod -l app=guitar-backend -o jsonpath='{.items[0].status.podIP}')
curl http://$POD_IP:2020/api/v1/health

# Check Fluent Bit's Prometheus metrics output
curl http://$POD_IP:2021/metrics
```

To verify logs are reaching Loki, open Grafana (http://192.168.100.201), go to **Explore**, select the Loki data source (you'll need to add it once at Settings → Data Sources → Loki → URL: `http://loki.default.svc.cluster.local:3100`), and run:

```logql
{job="guitar-backend"} |= "librosa"
```

## Resource Footprint

Fluent Bit is extremely lean — one of the leanest log processors available. The sidecar adds approximately 32MB of memory and negligible CPU under normal conditions, which is well within what k3master can absorb even at 8GB RAM.

| Container | Memory Request | Memory Limit | CPU Limit |
|-----------|---------------|--------------|-----------|
| guitar-backend | 256Mi | 1536Mi | 1000m |
| fluent-bit-sidecar | 32Mi | 64Mi | 100m |
| **Pod total** | **288Mi** | **1600Mi** | **1100m** |

## What's Next

The natural next step is building a Grafana dashboard that correlates the Prometheus memory metrics from Node Exporter (already running) with the log events from Loki — so you can see a memory spike and immediately jump to the logs from that exact moment in time, in the same UI. Grafana's Explore view supports this with the "Correlations" feature.

Adding an alert rule to fire when Fluent Bit detects the `librosa` OOM pattern in logs would complete the picture: automated detection → log evidence → metric context, all without touching the application code.
