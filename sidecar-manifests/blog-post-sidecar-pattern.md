# The Kubernetes Sidecar Pattern: Building Real Observability on a Phone Cluster

*Posted February 2026 — Part 4 of the HomeLab series*

---

When the guitar practice application started crashing silently, the debugging process was painfully manual. I'd notice a 502 error on the frontend, run `kubectl logs guitar-backend-xxxxx`, scroll through output looking for a timestamp that matched the crash, then cross-reference it with Grafana's memory graphs — two separate browser tabs, manual correlation, no automation. The root cause was known (librosa, the audio processing library, causes memory spikes on large files), but detecting *when* it happened and capturing the context around it required me to be watching at the right moment.

This post is about solving that problem using one of Kubernetes' most elegant patterns: the **sidecar**.

## What the Sidecar Pattern Actually Means

Before going into implementation, it's worth being precise about what a sidecar is, because the term is sometimes used loosely.

A Kubernetes **Pod** is not a container — it's a wrapper that can hold one or more containers that share certain resources. Specifically, every container in a pod shares the same network namespace (so they all see `localhost` as each other) and can optionally share mounted volumes. Each container still has its own filesystem root, its own process space, and its own resource limits. Think of a pod as an apartment building floor: each apartment (container) is separate, but all the apartments share the building's electrical system (network) and can share storage rooms (volumes) if you wire them that way.

A **sidecar** is simply a second container in that pod whose job is to help the first one — to extend or augment it — without being part of its core business logic. The name comes from motorcycle sidecars: the passenger car is attached to the bike, shares its momentum, but doesn't drive. It just handles what the driver can't do alone.

The sidecar pattern is powerful precisely because it achieves **separation of concerns at the infrastructure level**. The guitar-backend container doesn't need to know how to ship logs to Loki, format Prometheus metrics, or handle retry logic when the log store is temporarily unavailable. That's the sidecar's job. The application just writes a file.

## Why Not a Node-Level Log Collector?

The natural question is: "Couldn't you just run Fluentd as a DaemonSet on every node and collect logs from all containers centrally?" Yes, and that's exactly how many production clusters are configured. But it has a meaningful tradeoff.

A DaemonSet log collector reads **stdout from every container on the node**. That works well when your logs are unstructured text that you'll parse later. But the sidecar approach lets the application write a structured JSON file to a shared volume, which the sidecar reads and forwards *already parsed*. No regex, no guessing at timestamp formats, no ambiguity about which field is the error level. The structure is defined once, at the application level, and everything downstream benefits.

There's also a portability argument. If I move the guitar-backend pod to one of the OnePlus phone workers (which don't have a DaemonSet log collector running — and at 6–8GB RAM and ARM architecture, I'd rather not add one), the sidecar comes with the pod automatically. The observability capability is *baked into the deployment spec*, not dependent on what happens to be installed on a given node.

## The Shared Volume: The Key Mechanism

The entire pattern hinges on a Kubernetes `emptyDir` volume — one of the simplest volume types, and in this case exactly the right tool. An `emptyDir` volume is created when a pod starts, exists for the pod's entire lifetime, is deleted when the pod stops, and is visible to every container in that pod that mounts it.

In the pod spec, this looks like:

```yaml
volumes:
  - name: shared-logs
    emptyDir: {}
```

Then both containers mount it:

```yaml
# In guitar-backend container:
volumeMounts:
  - name: shared-logs
    mountPath: /var/log/guitar-backend

# In fluent-bit-sidecar container:
volumeMounts:
  - name: shared-logs
    mountPath: /var/log/guitar-backend
```

From each container's perspective, it's just a regular directory. The guitar-backend writes `app.log` there; Fluent Bit opens and tails the same file. The kernel handles the rest. There's no network call, no serialization, no protocol — just a file on a shared filesystem.

## Three Outputs From One Sidecar

The Fluent Bit configuration fans the log data out to three destinations simultaneously, which is what makes this a full-showcase implementation rather than a toy example.

**Loki** is the primary log storage backend. If you've used Prometheus, the mental model translates almost directly: instead of indexing time-series metric samples, Loki indexes *labels* (low-cardinality key-value pairs like `job=guitar-backend` or `node=k3master`) and stores the actual log text as compressed chunks. This makes it extremely storage-efficient and fast to query for the label-filtered use cases that matter most. Crucially, Grafana — already running at `192.168.100.201` — has native Loki support. Once Loki is deployed and added as a Grafana data source, you get a full log exploration UI in the same tool already used for metrics.

**Prometheus metrics** come from Fluent Bit's built-in `prometheus_exporter` output plugin. This exposes a `/metrics` endpoint on port 2021 of the pod, which Prometheus scrapes on its normal cycle. The metrics include things like records processed, bytes sent, and output plugin error rates — so you can alert on "Fluent Bit stopped shipping logs" as a distinct signal from "the application crashed."

**Stdout** remains available for real-time debugging. When you run `kubectl logs <pod> -c fluent-bit-sidecar`, you see every processed log record in human-readable form. This is invaluable during initial setup and when investigating a specific incident without opening Grafana.

## The Resource Story

One concern with adding a sidecar to every application pod is resource consumption. On k3master's 8GB RAM (with a 16GB upgrade planned), every megabyte counts. Fluent Bit is specifically designed for this constraint — it's written in C, the binary is around 450KB, and under normal conditions it consumes around 10–20MB of actual memory for this workload. The resource limits in the manifest (64Mi memory, 100m CPU) are deliberately generous to handle any burst, but in practice the headroom is large.

The pod's total resource footprint — guitar-backend plus the sidecar — is 288Mi requested memory and 1600Mi maximum. At that limit, the sidecar represents about 4% of the pod's memory ceiling, which is a reasonable price for the observability it provides.

## Deployment Order and Practical Notes

Because Fluent Bit is configured to ship logs to Loki, Loki must be running before the sidecar pod starts (or Fluent Bit will log connection errors and retry). The manifests are numbered to make the intended order explicit: ConfigMap first, then Loki deployment (with a rollout status wait), then the guitar-backend with its sidecar.

The most common issue I encountered during setup is that the guitar-backend was only logging to stdout, not to a file. The sidecar can only read what's in the shared volume, so the application needs a file handler added alongside its existing stdout handler. This is a three-minute Python change and doesn't require any restart of the existing production traffic during the transition — you can add file logging while keeping stdout, then verify the sidecar is reading correctly before removing the duplicate.

## Connecting Logs to Metrics: The Payoff

With this setup running, the crash investigation workflow transforms. When a librosa OOMKill happens, Prometheus captures the memory spike via Node Exporter (already deployed), Fluent Bit captures the log context — the filename being processed, the operation that triggered the spike, the stack trace if the app logged it — and both land in Grafana. Grafana's Explore view supports a feature called Correlations that lets you select a time range on a metric graph and jump directly to the Loki logs from that exact interval.

What was a manual, multi-tab, "I hope I was watching" process becomes a single Grafana investigation workflow. That's the concrete value of the pattern, beyond its architectural elegance.

---

*The full manifests and README for this pattern are available at [github.com/ivemcfire/ivemcfire.github.io](https://github.com/ivemcfire/ivemcfire.github.io). The previous posts in this series cover [building the k3s cluster on OnePlus phones](#), [deploying the guitar app](#), and [running Ollama locally with GPU acceleration](#).*
