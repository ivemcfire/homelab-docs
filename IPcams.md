# IP Cameras — k3master Homelab

Last updated: 2026-04-28

**Network (V14):** Cameras on flat `192.168.100.0/24` via Service Room Zyxel switch. VLAN 30 deferred to Phase 2+. Frigate compute target = k3node1 (i5-6600 + GTX 1050Ti) once Phase 2 onboarding complete.

---

## 1. Hardware

### Xiongmai (x7)

| Field | Value |
|-------|-------|
| Manufacturer | Hangzhou Xiongmai Technology Co., Ltd |
| Board/Module | IPG-50X10PT-S |
| Type | PTZ (Pan-Tilt-Zoom) IP Camera |
| Protocol | ONVIF |
| Units | 7 (cam11–16, cam18) |

### Reolink (x1)

| Field | Value |
|-------|-------|
| Manufacturer | Reolink |
| Type | 4K IP Camera |
| Protocol | ONVIF, RTSP (TCP only) |
| RTSP port | 554 |
| HTTPS port | 443 |
| ONVIF port | 8000 |
| Disabled ports | HTTP 80, Media 9000, RTMP 1935 |
| Face recognition | Enabled (Frigate 0.17 built-in) |
| Units | 1 (reolink @ .17) |

### Yi Home 720p (x1)

| Field | Value |
|-------|-------|
| Manufacturer | Yi (Xiaomi ecosystem) |
| Model | Yi Home 720p |
| Firmware | yi-hack (custom, RTSP enabled) |
| Type | Wi-Fi indoor camera |
| RTSP port | 554 (no auth) |
| RTSP restream port | 8554 |
| Telnet | Port 23 open (password unknown) |
| SSH | Disabled (port 22 closed) |
| HTTP | Port 80 (basic file listing, no config UI) |
| Connection | Wi-Fi (requires router gateway .1) |
| Units | 1 (yi-cam @ .19) |

---

## 2. Camera IP Map

| Camera | IP Address | Gateway | DNS | Status |
|--------|------------|---------|-----|--------|
| cam11 | 192.168.100.11 | 192.168.100.254 | 192.168.100.254 | Configured |
| cam12 | 192.168.100.12 | 192.168.100.254 | 192.168.100.254 | Configured |
| cam13 | 192.168.100.13 | 192.168.100.254 | 192.168.100.254 | Configured |
| cam14 | 192.168.100.14 | — | — | Pending config |
| cam15 | 192.168.100.15 | — | — | Pending config |
| cam16 | 192.168.100.16 | — | — | Pending config |
| reolink | 192.168.100.17 | 192.168.100.254 | 192.168.100.254 | Configured |
| cam18 | 192.168.100.18 | — | — | Pending config |
| yi-cam | 192.168.100.19 | 192.168.100.1 | — | Configured (Wi-Fi) |
| cam21 ANNKE | 192.168.100.21 | 192.168.100.254 | 192.168.100.254 | Configured (deployed off bench 2026-05-10) |
| cam22 ANNKE | 192.168.100.22 | 192.168.100.254 | 192.168.100.254 | Configured (Attic, deployed earlier) |
| cam23 ANNKE | 192.168.100.23 | — | — | Pending — same hardware as .21/.22 |
| cam24 ANNKE | 192.168.100.24 | — | — | Pending — same hardware as .21/.22 |

**Subnet:** 192.168.100.0/24
**IP range reserved for Xiongmai cameras:** .11–.18
**ANNKE C500 (5MP H.264, Hikvision-OEM):** .21–.24
**Reolink:** .17 | **Yi cam:** .19

---

## 3. Credentials

### Xiongmai

| Field | Value |
|-------|-------|
| Username | admin |
| Password | *(blank)* |
| Notes | Admin user cannot be changed or deleted (firmware limitation) |
| Web UI | http://192.168.100.XX |

### Reolink

| Field | Value |
|-------|-------|
| Username | admin |
| Password | <REOLINK_PASSWORD> |
| Web UI | https://192.168.100.17 (HTTPS only, HTTP disabled) |

---

## 4. Network Isolation

Cameras are LAN-only — no internet access.

| Setting | Value | Reason |
|---------|-------|--------|
| Gateway | 192.168.100.254 (dead IP) | No route to internet |
| DNS | 192.168.100.254 (dead IP) | No DNS resolution outside LAN |
| Default DNS (firmware) | 114.114.114.114 (ChinaNet) | Hardcoded by Xiongmai, overridden to .254 |

**Exception:** Yi cam (.19) uses router gateway (.1) — Wi-Fi requires valid gateway for association.

### Disabled Services

| Service | Status | Reason |
|---------|--------|--------|
| UPnP | Disabled | Prevents firewall hole-punching |
| PMS (cloud/P2P) | Disabled | No phone-home to Xiongmai cloud |
| Cloud | Disabled | No remote access via vendor cloud |
| FTP | Disabled | Frigate handles recording via NFS |
| Alarm Center | Disabled | Frigate handles detection |
| High Speed Download | Disabled | Interferes with RTSP stream stability |

---

## 5. Streaming

### RTSP

| Field | Value |
|-------|-------|
| Port | 554 |
| Authentication | Inline (user/password in URL) |

**URL format:**
```
Main:  rtsp://admin:@192.168.100.XX:554/user=admin&password=&channel=1&stream=0.sdp
Sub:   rtsp://admin:@192.168.100.XX:554/user=admin&password=&channel=1&stream=1.sdp
```

### Xiongmai Stream Specs (cam11 — reference)

| Stream | Resolution | Codec | Profile | FPS | Audio |
|--------|------------|-------|---------|-----|-------|
| Main (stream=0) | 1280x720 | H.264 | Main | 25 | None |
| Sub (stream=1) | 704x576 | H.264 | Main | 5 | None |

### Reolink Stream Specs

| Stream | Resolution | Codec | Profile | FPS | Bitrate | Audio |
|--------|------------|-------|---------|-----|---------|-------|
| Main | 3840x2160 (4K) | H.265 (HEVC) | Main | 25 | 6144 kbps | AAC mono 16kHz |
| Sub | 640x360 | H.264 | High | 15 | 512 kbps | AAC mono 16kHz |

> **Sub stream limitation:** 640x360 is the **maximum and only** resolution option.
> FPS range: 7–15 (set to 15). Bitrate range: 256–512 kbps (set to 512).
> Sub stream is NOT used for Frigate detection — too low resolution for face recognition.
> Main 4K stream is used for both detect + record; Frigate downscales to 720p for detection.

**Reolink RTSP URL format:**
```
Main:  rtsp://admin:<REOLINK_PASSWORD>@192.168.100.17:554/h264Preview_01_main
Sub:   rtsp://admin:<REOLINK_PASSWORD>@192.168.100.17:554/h264Preview_01_sub
```
**Reolink requires `rtsp_transport: tcp`** — rejects UDP.

### Yi Stream Specs

| Stream | Resolution | Codec | Profile | FPS | Audio |
|--------|------------|-------|---------|-----|-------|
| Main (ch0_0) | 1280x720 | H.264 | High | ~20 | AAC stereo 8kHz |
| Sub (ch0_1) | 640x360 | H.264 | High | ~20 | AAC stereo 8kHz |

**Yi RTSP URL format (no auth):**
```
Main:  rtsp://192.168.100.19:554/ch0_0.h264
Sub:   rtsp://192.168.100.19:554/ch0_1.h264
```
**Yi requires `rtsp_transport: tcp`.**

### Optimized Settings

| Setting | Camera | Value | Reason |
|---------|--------|-------|--------|
| Network Transmission Policy | Xiongmai | Fluency | Stable stream for Frigate decode |
| Sub stream FPS | Xiongmai/Yi | 5 | Frigate detection only needs 5fps |
| Sub stream FPS | Reolink | 15 (max) | Not used for detect, but available for go2rtc restream |
| Sub stream bitrate | Reolink | 512 kbps (max) | Reduces compression artifacts on faces |
| UPnP | All | Disabled | No firewall hole-punching |
| NTP | Reolink | Disabled | Uses .254 dead gateway, can't reach NTP |

---

## 6. Frigate Integration

| Field | Value |
|-------|-------|
| Status | **Deployed and running** |
| NVR | Frigate 0.17.2 (`ghcr.io/blakeblackshear/frigate:0.17.1`) |
| Namespace | `frigate` |
| MetalLB IP | 192.168.100.209 (Web UI :5000, RTSP restream :8554) |
| Node | k3master (amd64 — CPU video decode) |
| MQTT | Mosquitto at 192.168.100.207:1883 |
| Storage — primary | Local 1.8TB HDD on k3master (`/dev/sda1` → `/mnt/frigate`) |
| Storage — config DB | hostPath `/mnt/frigate/config` on k3master |
| Storage — backup | Jumphost 3.6TB USB, nightly rsync mirror (CronJob `frigate-backup` at 2 AM) |
| Detection engine | CPU-based TFLite, 4 threads, ~28ms inference (no Coral TPU) |
| HW accel | Intel QSV (Ice Lake i5-1035G1) — H.264/H.265 decode |
| Face recognition | Enabled on Reolink only (`min_area: 50`, small model, built-in Frigate 0.17) |
| Recording mode | Event-only: 5s pre-capture, 10s post-capture |
| Retention | **90 days** (clips + snapshots) |
| Internet access | `https://cams.ayurforlife.eu` via Cloudflare Tunnel + Access (email OTP) |
| Manifests | `~/frigate-k8s/` |
| Monitoring | Alertmanager → Telegram, Frigate exporter → Prometheus, Grafana dashboard |
| Monitoring manifests | `~/monitoring-stack/` |
| Grafana dashboard | `http://192.168.100.201/d/frigate-nvr/frigate-nvr` |
| Alerts | 16 PrometheusRules: camera down, backup failed, storage low, node health, etc. |

### Cloudflare Tunnel

| Field | Value |
|-------|-------|
| Tunnel name | `frigate-tunnel` |
| Tunnel ID | `471b5cdf-82cd-430c-a275-c3068c50eaa4` |
| Hostname | `cams.ayurforlife.eu` |
| Auth | Cloudflare Access — email OTP, allowed: `ayurforlife@gmail.com`, 24h sessions |
| Deployment | cloudflared pod in frigate namespace, pinned to k3master |
| Manifest | `~/frigate-k8s/cloudflared.yaml` |

### Active Cameras in Frigate

| Frigate name | Camera | Detect | Record | Frigate native face_rec | Double Take → CompreFace |
|---|---|---|---|---|---|
| cam01_front_yard | Xiongmai .11 | Sub 640x480 @ 5fps | Main 720p @ 25fps | No | Not in DT cameras list |
| cam02 | Xiongmai .12 | Sub 640x480 @ 5fps | Main 720p @ 25fps | No | Not in DT cameras list |
| cam03 | Xiongmai .13 | Sub 640x480 @ 5fps | Main 720p @ 25fps | No | Not in DT cameras list |
| cam04–06 | Xiongmai .14–.16 | — | — | No | — (cam14 disabled, replaced by cam21; cam15/.16 active no DT) |
| cam07_reolink | Reolink .17 | **Main 4K → 1280x720 @ 10fps** | Main 4K H.265 @ 25fps | No (was Yes, off post-purge) | **Yes — snapshot.attempts=5** (low event count due to area traffic, not a HW issue) |
| cam08_yicam | Yi .19 | Sub 640x480 @ 5fps | Main 720p @ 20fps | No | Not in DT cameras list |
| cam22 | ANNKE C500 .22 (Attic) | Main 1920x1080 @ 5fps (TCP) | Main 5MP H.264 @ ? fps | No | **Yes — snapshot.attempts=5** (cam network DOWN as of 2026-05-10) |
| cam21 | ANNKE C500 .21 (Bench-deployed 2026-05-10) | Main 1920x1080 @ 5fps (TCP) | Main 5MP H.264 @ ? fps | No (toggled on→off 2026-05-10, DT path chosen) | **Yes — snapshot.attempts=5** (added 2026-05-10 mirroring cam22) |

> **DT path:** Frigate person event → MQTT topic frigate/events → Double Take → snapshot pull → CompreFace k3face app (key wired). Matches stored at /mnt/frigate/double-take/matches/<subject>/. CompreFace UI: http://192.168.100.212/. Reference photos must be enrolled per subject before any match works (5 subjects exist with 0 images post-purge).

### Event filter helpers (on k3master, ~/bin/)

Three CLI scripts on k3master (in PATH after ~/bin: prepend) for browsing Frigate events by face recognition status:

| Command | What it does |
|---------|--------------|
| frigate-strangers [limit] | Lists recent events with no sub_label (DT didn't recognize a known face — likely strangers). |
| frigate-subject <name> [limit] | Lists events where DT matched the named subject (e.g. frigate-subject Ivalin). |
| frigate-sublabels [limit] | Distribution count of all sub_labels across recent events (sanity check for DT push-back). |

Sub_labels are populated by Double Take when it matches a face with confidence ≥ 70 (per detect.match.confidence in DT config). Strangers and unknown faces get no sub_label, so frigate-strangers is the natural feed for browse non-family clips. Frigate UI search box also accepts sub_labels:Ivalin or similar.

> **cam07 note:** Uses main 4K stream for both detect and record. Frigate downscales 4K to 720p
> for detection — preserves native pixel detail for face recognition. Sub stream (640x360) is
> too low resolution. Higher CPU usage than other cameras due to 4K H.265 decode.

### Face Recognition Lessons Learned

1. **Upscaling doesn't help.** Setting detect to 1280x720 on a 640x360 sub stream just stretches pixels — no new detail.
2. **Downscaling preserves detail.** 4K main → 720p detect retains sharp facial features.
3. **go2rtc transcode is CPU-heavy.** `ffmpeg:cam07#video=h264#width=1280#height=720` caused 80-90% CPU. Direct 4K to Frigate is better.
4. **Lowering thresholds is not a fix.** Reduces confidence requirements without improving input quality.
5. **Training data required.** Models loaded (`facedet.onnx`, `facenet.tflite`) but need reference photos via Frigate UI.
6. **`face` is not a valid track object.** Default TFLite model uses COCO classes — face recognition is a separate pipeline on person detections.
7. **`snapshots` is not a valid ffmpeg input role.** Only `record`, `detect`, and `audio` are valid in Frigate 0.17.

---

## 7. Per-Camera Configuration Checklist

Apply to each camera during setup:

- [ ] Set static IP (192.168.100.1X)
- [ ] Set gateway to 192.168.100.254
- [ ] Set DNS to 192.168.100.254
- [ ] Set network transmission policy to Fluency
- [ ] Disable High Speed Download
- [ ] Disable UPnP
- [ ] Disable PMS / Cloud
- [ ] Disable Alarm Center
- [ ] Disable FTP
- [ ] Enable RTSP on port 554
- [ ] Set sub stream FPS to 5
- [ ] Verify main stream via ffprobe (stream=0)
- [ ] Verify sub stream via ffprobe (stream=1)

### Verification command:
```bash
# Main stream
ffprobe -v quiet -print_format json -show_streams \
  "rtsp://admin:@192.168.100.XX:554/user=admin&password=&channel=1&stream=0.sdp"

# Sub stream
ffprobe -v quiet -print_format json -show_streams \
  "rtsp://admin:@192.168.100.XX:554/user=admin&password=&channel=1&stream=1.sdp"
```
