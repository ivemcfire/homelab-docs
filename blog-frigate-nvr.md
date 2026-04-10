# Running Frigate NVR on Kubernetes: IP Cameras, Face Recognition, and a 1.8TB Hard Drive

**Turning nine cheap IP cameras into a self-hosted surveillance system on k3s — with Cloudflare Tunnel for remote access, nightly backup to a netbook, and lessons learned about why upscaling a 640x360 stream doesn't magically create detail.**

---

## The Problem

I had nine IP cameras — seven Xiongmai PTZ units, a Reolink 4K, and a Yi Home 720p — all sitting on the network doing nothing useful. Each one had its own web UI, its own storage settings, its own motion detection that either missed everything or fired constantly. Checking footage meant opening a browser tab per camera, scrubbing through SD card recordings in a clunky firmware interface, and hoping the event I cared about was on the right camera.

No unified timeline. No object detection. No way to pull up "show me every time a person was on cam01 last Tuesday" without watching hours of footage manually.

What I wanted was a single system that could ingest every camera stream, detect objects in real time, record only when something interesting happens, and let me search by event type. Frigate NVR does exactly that — and it runs as a container, which means it fits into the k3s cluster I already have.

This post documents the full deployment: camera hardening, stream configuration, Kubernetes manifests, face recognition on the Reolink, Cloudflare Tunnel for remote access, and the backup strategy.

---

## The Camera Fleet

Nine cameras, three different manufacturers, three different quirks.

### Xiongmai (x7) — cam11 through cam16, cam18

These are budget PTZ cameras built on the IPG-50X10PT-S module. ONVIF-compatible, 720p main stream, H.264, 25fps. The sub stream runs at 704x576 but I drop it to 5fps for detection — Frigate doesn't need more than that for object tracking.

The RTSP URLs are verbose:

```
rtsp://admin:@192.168.100.XX:554/user=admin&password=&channel=1&stream=0.sdp   # Main
rtsp://admin:@192.168.100.XX:554/user=admin&password=&channel=1&stream=1.sdp   # Sub
```

The admin password is blank and cannot be changed — firmware limitation. Security comes from network isolation instead (more on that below).

### Reolink (x1) — cam07

The flagship camera. 4K (3840x2160) main stream at 25fps, H.265, 6144 kbps with AAC audio. This is the one running face recognition. RTSP over TCP only — it rejects UDP connections.

```
rtsp://admin:<REOLINK_PASSWORD>@192.168.100.17:554/h264Preview_01_main
```

The sub stream caps out at 640x360. That number matters — it's why I ended up feeding the full 4K stream to Frigate and letting it downscale, rather than using the sub stream for detection like every other camera.

### Yi Home 720p (x1) — cam08

A Wi-Fi indoor camera running yi-hack custom firmware to expose RTSP. No authentication on the stream, telnet wide open on port 23. The one camera that actually needs a valid gateway to function because Wi-Fi association requires it.

```
rtsp://192.168.100.167:554/ch0_0.h264   # Main
rtsp://192.168.100.167:554/ch0_1.h264   # Sub
```

---

## Network Isolation

These cameras are LAN-only. They cannot reach the internet.

The trick is simple: set the gateway and DNS to a dead IP address. I use `192.168.100.254` — nothing lives there. The cameras can talk to the local subnet (where Frigate lives) but have no route out. No DNS resolution, no cloud callbacks, no firmware phoning home.

This matters because the Xiongmai cameras ship with `114.114.114.114` (ChinaNet public DNS) hardcoded as the default DNS server, and PMS (their peer-to-peer cloud service) enabled out of the box. Left alone, these cameras will happily punch through your firewall via UPnP and phone home to servers in China.

Every camera gets the same lockdown:

| Setting | Status | Reason |
|---------|--------|--------|
| UPnP | Disabled | Prevents firewall hole-punching |
| PMS / Cloud | Disabled | No phone-home to vendor cloud |
| FTP | Disabled | Frigate handles all recording |
| Alarm Center | Disabled | Frigate handles detection |
| High Speed Download | Disabled | Destabilizes RTSP streams |

The Yi cam is the exception — it uses the real router gateway at `.1` because Wi-Fi association won't complete without it. It's the only camera that could theoretically reach the internet, though firewall rules block it.

---

## Frigate on Kubernetes

Frigate 0.17.2 runs on k3master, the amd64 laptop that serves as the K3s control plane. It has to run here — the phone workers (OnePlus 6/6T running postmarketOS) don't have the CPU headroom for real-time video decode of nine camera streams.

The key specs:

| Field | Value |
|-------|-------|
| Image | `ghcr.io/blakeblackshear/frigate:0.17.1` |
| Namespace | `frigate` |
| MetalLB IP | 192.168.100.211 |
| Detection | CPU-based TFLite, 2 threads |
| MQTT | Mosquitto at 192.168.100.207:1883 |
| Storage | 1.8TB local HDD (`/dev/sda1` mounted at `/mnt/frigate`) |
| Retention | 90 days (clips + snapshots) |

No Coral TPU. Detection runs on CPU, which is fine for the current camera count. The TFLite model uses standard COCO classes — person, car, dog, cat, and so on.

### Camera Configuration in Frigate

Most cameras follow the same pattern: sub stream for detection, main stream for recording. Frigate only needs to run inference on the lower-resolution stream — when it detects something, it pulls the high-res recording from the main stream.

| Camera | Detect stream | Record stream | Face recognition |
|--------|--------------|---------------|-----------------|
| cam01–cam03 (Xiongmai) | Sub 640x480 @ 5fps | Main 720p @ 25fps | No |
| cam04–cam06 (Xiongmai) | Disabled | Disabled | No |
| cam07_reolink | **Main 4K downscaled to 720p @ 10fps** | Main 4K H.265 @ 25fps | **Yes** |
| cam08_yicam (Yi) | Sub 640x480 @ 5fps | Main 720p @ 20fps | No |

cam07 is the outlier. It uses the main stream for _both_ detect and record, because the sub stream is too low-resolution for face recognition. More on why below.

---

## Face Recognition: Why Resolution Matters More Than Settings

Frigate 0.17 ships with built-in face recognition — `facedet.onnx` for detection and `facenet.tflite` for embedding. I enabled it on the Reolink with `min_area: 500` and expected it to work out of the box on the sub stream.

It didn't.

The Reolink's sub stream maxes out at 640x360. That's the only option — there's no way to increase it in the camera's settings. At that resolution, faces are typically 20-30 pixels wide, which is not enough for the embedding model to produce useful vectors.

My first instinct was to upscale. I configured go2rtc to transcode the sub stream to 1280x720:

```
ffmpeg:cam07#video=h264#width=1280#height=720
```

This was wrong for two reasons. First, upscaling a 640x360 source to 720p just stretches the same pixels over a larger area. No new detail is created. A blurry face at 30 pixels becomes a blurry face at 60 pixels — the embedding model still can't find features that aren't there. Second, the go2rtc transcode pinned the CPU at 80-90%. It was decoding H.265 4K, scaling to 720p, and re-encoding to H.264, all in software.

The solution was counterintuitive: feed the full 4K main stream directly to Frigate and let it downscale to 720p for detection. Downscaling _preserves_ native pixel detail — a face that's 120 pixels wide at 4K becomes 60 pixels at 720p, but those 60 pixels contain real high-frequency detail from the original sensor. Much better input for the embedding model than 60 interpolated pixels from an upscaled 360p stream.

The CPU cost of decoding 4K H.265 at 10fps is meaningful but manageable — significantly less than the go2rtc transcode approach, and the detection quality is dramatically better.

### Lessons the Hard Way

A few more things I learned by getting them wrong first:

1. **`face` is not a valid object class.** The default TFLite model uses COCO classes. Face recognition is a separate pipeline that runs _after_ a `person` detection — you don't track faces directly.
2. **`snapshots` is not an ffmpeg input role.** Only `record`, `detect`, and `audio` are valid in Frigate 0.17. I spent time debugging a config that looked right but silently did nothing.
3. **Lowering confidence thresholds doesn't fix bad input.** If the source resolution can't resolve facial features, accepting lower-confidence matches just produces more false positives.
4. **Training data is required.** The models ship ready, but without reference photos uploaded through the Frigate UI, there's nothing to match against. Detection works, but identification needs examples.

---

## Remote Access via Cloudflare Tunnel

Frigate's web UI runs on port 5000 at 192.168.100.211 — reachable on the LAN but not from outside. I didn't want to port-forward or expose the k3master directly, so I set up a Cloudflare Tunnel.

A `cloudflared` pod runs in the frigate namespace, pinned to k3master alongside Frigate itself. It establishes an outbound connection to Cloudflare's edge and maps `cams.ayurforlife.eu` to `http://frigate-service:5000` inside the cluster.

Authentication uses Cloudflare Access with email OTP — only `ayurforlife@gmail.com` can log in, and sessions expire after 24 hours. No VPN, no open ports, no dynamic DNS.

The result: I can pull up the Frigate dashboard from my phone on any network by navigating to `https://cams.ayurforlife.eu`, entering my email, and pasting the one-time code. Live streams, event history, face recognition results — all available remotely through a zero-trust tunnel.

---

## Backup Strategy

All Frigate data — recordings, snapshots, the config database — lives on a 1.8TB HDD physically attached to k3master. That's a single point of failure.

The backup is a Kubernetes CronJob called `frigate-backup` that runs at 2:00 AM every night. It rsyncs the entire `/mnt/frigate` directory to a 3.6TB USB drive connected to the jumphost at 192.168.100.152. The jumphost is a netbook that also serves as the SSH bastion for the cluster.

The rsync is a mirror — deletions propagate. If Frigate's retention policy purges a 91-day-old clip from the primary, the next backup run removes it from the jumphost too. Both drives stay in sync, and neither grows unbounded.

With 90-day retention and event-only recording (5 seconds pre-capture, 10 seconds post-capture), storage growth is modest. Continuous 24/7 recording from nine cameras would fill the drive in weeks; event-only mode means most of the time, nothing is being written.

---

## Things That Went Wrong

**ONVIF PTZ doesn't work on this Reolink.** The camera advertises ONVIF support, but PTZ controls through Frigate's ONVIF integration return errors. Pan and tilt work through the Reolink web UI and app, but not through the standard protocol. I haven't investigated further — PTZ control from Frigate would be nice but isn't critical.

**go2rtc transcode was a trap.** It seemed like the right approach — use go2rtc as a restream proxy, transcode to a friendlier resolution. But the CPU cost was brutal (80-90% sustained) and the quality gain was zero because upscaling doesn't add information. Removing the transcode and feeding native 4K directly to Frigate was both cheaper and better.

**The Yi cam needs the real gateway.** I initially configured it with the `.254` dead gateway like every other camera. Wi-Fi association failed silently — the camera just dropped offline. The Yi needs a valid route to the access point's gateway IP for the Wi-Fi handshake to complete.

**Xiongmai "High Speed Download" breaks streams.** This is a firmware feature that sounds helpful but interferes with RTSP stability. With it enabled, streams would drop and reconnect every few minutes. Disabling it fixed the issue immediately.

---

## The Payoff

Before Frigate, checking cameras meant nine browser tabs and manual scrubbing. Now I have a single dashboard showing real-time object detection across every active camera, event-based recording with searchable history, face recognition on the 4K camera, and remote access from anywhere through a Cloudflare tunnel — all running on a Kubernetes cluster where the worker nodes are old phones.

The total cost of the camera hardware was well under what a single commercial NVR system would run, and the software is open source. The infrastructure was already there — k3master had spare CPU, the HDD was sitting unused, and the Cloudflare free tier covers the tunnel.

Most importantly, it's observable. Frigate publishes events to MQTT, which means I can wire up alerts, dashboards, or automations through the same Mosquitto broker that the HydroFlow IoT project already uses. The cameras went from being nine isolated devices to being part of the cluster.

---

## Cluster Context

This runs on the same k3s cluster described in previous posts — a Lenovo laptop as the control plane and three OnePlus smartphones (6, 6, 6T) running postmarketOS as ARM64 workers, connected via USB ethernet.

| Node | Role | Arch | Hardware |
|------|------|------|----------|
| k3master | Control plane + Frigate | amd64 | x86_64 laptop (headless) |
| one6t | Worker | arm64 | OnePlus 6T (Snapdragon 845, 8GB) |
| one62 | Worker | arm64 | OnePlus 6 (Snapdragon 845, 6GB) |
| one61 | Worker | arm64 | OnePlus 6 (Snapdragon 845, 6GB) |

Frigate is pinned to k3master — video decode is too CPU-heavy for the phone nodes. Everything else in the cluster (Postgres, Gitea, Mosquitto, application backends) runs on the phones.

Manifests are in `~/frigate-k8s/`. Full infrastructure reference in `INFRA.md`.

---

*Previous posts: [The Kubernetes Sidecar Pattern](sidecar-pattern.html) · More at [ivemcfire.github.io](https://ivemcfire.github.io)*
