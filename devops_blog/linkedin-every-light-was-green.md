The Grafana ran in the cluster. The Uptime Kuma ran in the cluster. The Alertmanager paged Telegram through the cluster's network. When the cluster fell over, nothing was watching — and the dashboards that should have told me said everything was fine.

Three days of homelab observability work where every layer of the monitoring stack was a tenant of the thing it was supposed to observe.

The friction:

→ Grafana pinned to k3master via nodeSelector — single-node outage took the dashboard down with it
→ Uptime Kuma running inside the namespace it was meant to watch
→ Alertmanager → Telegram via cluster egress — bot unreachable exactly when paging matters
→ Tailscale --accept-routes hijack on .62 (sibling of yesterday's .52 fix) — cluster→jumphost ssh timed out for ~30 h
→ ChickenFlow postgres-backup with no nodeSelector — half the runs landed on phone nodes where in-pod DNS doesn't reach CoreDNS
→ CPUThrottlingHigh on every node-exporter — chart's 200m CPU limit vs scrape burst, 52–85% throttle, useless alert

The fixes:

→ Grafana → 11.5.2 in Docker on the jumphost, new prometheus-lan LB, dashboards source-controlled
→ In-cluster Uptime Kuma deleted, .62 Docker Kuma exposed at uptime.ayurforlife.eu
→ monitor.ayurforlife.eu behind Cloudflare Access SSO, same gate as cams.*
→ Alertmanager: Telegram → Gmail SMTP, no cluster-network dependency
→ tailscale set --accept-routes=false on .62 — 30 h of stuck postgres-backup CronJobs recovered
→ node-exporter CPU limit removed: CFS throttle 52–85% → 0%

Real observability is not a stack of tools — it is a discipline of placing each watcher one layer outside what it watches.

https://ivemcfire.github.io/posts/every-light-was-green.html
https://github.com/ivemcfire/homelab-config
