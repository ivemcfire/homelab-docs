I was away from the house when the cameras started returning 502. Then monitor.ayurforlife.eu did the same. By the time I got home, the cluster had healed itself and the dashboard said everything was fine — which, in real life, is when you start to worry.

The honest part of the story is that nothing was fine. One cloudflared pod was carrying every public service, pinned to one node, and that one node had its moment. Cloudflare's protocol supported HA, my Deployment did not. The next day, when I sat down to fix it properly, the cluster kept teaching me — Tailscale silently hijacking LAN routes, coredns forwarding into a black hole inside the pod netns, a Frigate pod with no CPU limit about to eat its own node. Each layer was claiming "fine" against a different definition of fine.

The friction:

→ One cloudflared connector for the whole tunnel — first node blip was total outage
→ coredns Corefile forwarding via /etc/resolv.conf — host's 127.0.0.53 stub is not reachable from inside a pod's netns
→ Tailscale subnet router accepting its own advertised LAN route — peer traffic pulled into the tunnel and DERP-relayed
→ Frigate pod with no CPU limit — 8-camera detect storm absorbed all 4 cores, kubelet starved, full-node lockup
→ k3master left with no default route and empty [Resolve] after the Tailscale install — pods stuck in ImagePullBackOff

The fixes:

→ cloudflared 1 → 3 replicas, PDB minAvailable: 2, control-plane-only nodeAffinity
→ Tailscale: 2 redundant subnet routers, --accept-routes=false on every LAN-resident node
→ coredns: forward . 1.1.1.1 8.8.8.8 — hardcoded, not host-derived
→ Frigate pod: CPU limit 3 cores, mem 4–6 Gi, shm 512 Mi
→ k3frigate kubelet: kube-reserved + system-reserved + 4 GB zram
→ Full-node lockup events: 2 in 48 h → 0 since hardening

High availability stops being a feature you add — and becomes the absence of the habit of trusting your own dashboards.

https://ivemcfire.github.io/posts/cloudflared-ha-and-oob.html
https://github.com/ivemcfire/homelab-config
