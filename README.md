# Homelab Documentation

Documentation, architecture notes, blog drafts, and project references for a bare-metal K3s homelab cluster running on an Intel i5-1035G1 control plane with 3x ARM64 phone workers (OnePlus 6/6T, postmarketOS).

## Index

Start with [master-v17.5.md](master-v17.5.md) (current architecture) and [cluster-improvements.md](cluster-improvements.md) (dated changelog of all infra work).

## Structure

```
homelab-docs/
  master-v17.5.md                # Master architecture doc (current; v17.3/v17.4 kept as history)
  cluster-improvements.md        # Dated changelog of infra work
  RECOVERY.md                    # Disaster-recovery runbook (age keys, rebuild order)
  info.md                        # k3master hardware, OS, software reference
  IPcams.md                      # Camera hardware specs, RTSP URLs, codecs
  phase2-plan.md                 # HA migration plan (completed 2026-05-12/13)
  project_ideas.md               # GPU compute project ideas
  blog-frigate-nvr.md            # Blog: Frigate NVR on K3s
  jumphost-fix-runbook.md        # Incident runbooks (also: 2026-06-07-usb-power-cascade.md,
  lb-guitar-backend-fix.md       #   lb-guitar-backend-fix.md)
  devops_blog/                   # Blog + LinkedIn post sources (published at ivemcfire.github.io)
  hydroflow/                     # HydroFlow IoT project docs
  ivmos-guitar-practice-app/     # Guitar practice app docs
  monitoring-stack/              # Monitoring notes
```

## Related Repos

| Repo | Description |
|------|-------------|
| [homelab-config](https://github.com/ivemcfire/homelab-config) | Cluster IaC — single source of truth for all manifests (consolidated 2026-07-12) |
| [frigate-k8s](https://github.com/ivemcfire/frigate-k8s) | ARCHIVED 2026-07-12 — manifests folded into homelab-config apps/frigate/ |
| [gpu-worker](https://github.com/ivemcfire/gpu-worker) | YOLOv8 inference on ARM64 phone GPUs via ONNX Runtime + OpenCL |
| [chickenFlow](https://github.com/ivemcfire/chickenFlow) | Angular 21 + ESP32-S2 chicken coop automation |

## License

MIT
