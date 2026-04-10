# Homelab Documentation

Documentation, architecture notes, blog drafts, and project references for a bare-metal K3s homelab cluster running on an Intel i5-1035G1 control plane with 3x ARM64 phone workers (OnePlus 6/6T, postmarketOS).

## Index

See [library.md](library.md) for the full documentation index.

## Structure

```
homelab-docs/
  library.md                     # Master index of all documentation
  info.md                        # k3master hardware, OS, software reference
  IPcams.md                      # Camera hardware specs, RTSP URLs, codecs
  project_ideas.md               # GPU compute project ideas (reviewed 3 rounds)
  blog-frigate-nvr.md            # Blog: Frigate NVR on K3s
  devops_blog/
    blog-edge-ai-phones.md       # Blog: Edge AI on recycled phones
  hydroflow/
    README.md                    # HydroFlow IoT project overview
    CONTEXT.md                   # Architecture decisions and deployment status
    backend/
      handover.md                # API routes, DB schema, environment variables
  ivmos-guitar-practice-app/
    README.md                    # Guitar practice app overview
    frontend/
      README.md                  # Frontend details
  sidecar-manifests/
    README.md                    # Fluent-bit sidecar logging pattern
    blog-post-sidecar-pattern.md # Blog reference on K8s sidecar pattern
```

## Related Repos

| Repo | Description |
|------|-------------|
| [frigate-k8s](https://github.com/ivemcfire/frigate-k8s) | Frigate NVR K8s manifests — write-back sidecar, RBAC, secret management |
| [gpu-worker](https://github.com/ivemcfire/gpu-worker) | YOLOv8 inference on ARM64 phone GPUs via ONNX Runtime + OpenCL |

## License

MIT
