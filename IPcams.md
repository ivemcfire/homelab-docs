# IP Cameras — k3master Homelab

Last updated: 2026-04-28

**Network (V14):** Cameras on flat `192.168.100.0/24` via Service Room Zyxel switch. VLAN 30 deferred to Phase 2+. Frigate compute target = k3node1 (i5-6600 + GTX 1050Ti) once Phase 2 onboarding complete.

---

## 1. Hardware

### Xiongmai (x5 active + 1 retired)

| Field | Value |
|-------|-------|
| Manufacturer | Hangzhou Xiongmai Technology Co., Ltd |
| Board/Module | IPG-50X10PT-S |
| Type | PTZ (Pan-Tilt-Zoom) IP Camera |
| Protocol | ONVIF |
| Units | 5 active (cam11, cam12, cam13, cam15, cam16) + 1 retired (cam14, replaced 2026-05-10) — cam18 reassigned to Reolink |

### Reolink (x2)

| Field | Value |
|-------|-------|
| Manufacturer | Reolink |
| Model | RLC-812A (`IPC_523B188MP`), fw `v3.1.0.920_2402207844` |
| Type | 4K-capable IP camera — main stream run at 2560x1440 H.264 (see Reolink Stream Specs) |
| Protocol | ONVIF, RTSP (TCP only) |
| RTSP port | 554 |
| HTTPS port | 443 |
| ONVIF port | 8000 |
| Disabled ports | HTTP 80, Media 9000, RTMP 1935 |
| Face recognition | Enabled (Frigate 0.17 built-in) |
| Units | 2 (cam17 @ .17, cam18 @ .18 — same RLC-812A model) |

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

### ANNKE C500 (x4)

| Field | Value |
|-------|-------|
| Manufacturer | ANNKE (Hikvision OEM) |
| Model | C500 (5MP H.264 turret, fixed lens, NO PTZ) |
| Protocol | ONVIF, RTSP (TCP) |
| RTSP port | 554 |
| ONVIF port | 8000 (ANNKE/Hikvision-OEM default; some firmwares 80) |
| HTTP port | 80 |
| ONVIF status | **Enabled** on cam web UI (confirmed 2026-05-11) |
| ONVIF auth | Digest, same admin creds as web UI |
| WS-Discovery | UDP 3702 — cams respond to subnet probes |
| Cam-side analytics | Available (motion, line-cross, intrusion) — **DO NOT WIRE**. Entry-level cam logic = high FP rate. Frigate detector path superior. Leave OFF. |
| ONVIF event subscription | Available — **DO NOT WIRE** to HA / Frigate / MQTT. Architectural bloat, no signal gain over Frigate. |
| Probing | Skip onvif-cli / wsdd unless troubleshooting connectivity. RTSP feed sufficient. |
| Face recognition | Frigate 0.17 native, enabled on cam21 + cam22. (External CompreFace/Double Take stack retired 2026-06 — not replaced, superseded.) |
| Units | 4 (cam21–24; .21 deployed off-bench, .22 Attic, .23/.24 pending) |

---

## 2. Camera IP Map

| Camera | IP Address | Gateway | DNS | Status |
|--------|------------|---------|-----|--------|
| cam11 | 192.168.100.11 | 192.168.100.254 | 192.168.100.254 | Configured |
| cam12 | 192.168.100.12 | 192.168.100.254 | 192.168.100.254 | Configured |
| cam13 | 192.168.100.13 | 192.168.100.254 | 192.168.100.254 | Configured |
| cam14 | 192.168.100.14 | — | — | **RETIRED 2026-05-10** (Xiongmai, replaced — see cluster-improvements row 38) |
| cam15 | 192.168.100.15 | — | — | Pending config |
| cam16 | 192.168.100.16 | — | — | Pending config |
| reolink | 192.168.100.17 | 192.168.100.254 | 192.168.100.254 | Configured |
| cam18 | 192.168.100.18 | 192.168.100.254 | 192.168.100.254 | Reolink RLC-812A (same model as cam17) — configured; **set main to 2560x1440 H.264 before adding to go2rtc** |
| yi-cam | 192.168.100.19 | 192.168.100.1 | — | Configured (Wi-Fi) |
| cam21 ANNKE | 192.168.100.21 | 192.168.100.254 | 192.168.100.254 | Configured (deployed off bench 2026-05-10) |
| cam22 ANNKE | 192.168.100.22 | 192.168.100.254 | 192.168.100.254 | Configured (Attic, deployed earlier) |
| cam23 ANNKE | 192.168.100.23 | — | — | Pending — same hardware as .21/.22 |
| cam24 ANNKE | 192.168.100.24 | — | — | Pending — same hardware as .21/.22 |

**Subnet:** 192.168.100.0/24
**IP range Xiongmai:** .11–.16 (.14 retired) | **Reolink RLC-812A:** .17–.18 | **ANNKE C500:** .21–.24 | **Yi:** .19

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
| Web UI | https://192.168.100.17 (HTTPS only, HTTP disabled) — also covers cam18 (same hardware) |

### ANNKE

| Field | Value |
|-------|-------|
| Username | admin |
| Password | **Same as Reolink admin password** (verified 2026-05-12 in frigate-config cm) |
| Web UI | http://192.168.100.2[1-4] |
| ONVIF auth | Same as web UI (admin + password) |
| K8s secret | `frigate-rtsp-creds` in `frigate` ns, key `FRIGATE_REOLINK_PASSWORD` — **misnomer**: var name predates ANNKE addition, key serves both Reolink (.17/.18) and ANNKE (.21–.24). Rename to `FRIGATE_RTSP_PASSWORD` on next refactor pass. |

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
| Main | 2560x1440 | H.264 | High | 25 | 8192 kbps | AAC mono 16kHz |
| Sub | 640x360 | H.264 | High | 15 | 512 kbps | AAC mono 16kHz |

> **Sub stream limitation:** 640x360 is the **maximum and only** resolution option.
> FPS range: 7–15 (set to 15). Bitrate range: 256–512 kbps (set to 512).
> Sub stream is NOT used for Frigate detection — too low resolution for face recognition.
> Main stream is used for **record**; detect uses the `cam17_detect` go2rtc transcode (1280x720).

> **Main stream codec — changed 2026-07-25 (H.265 -> H.264).** Chrome on Linux ships no software
> HEVC decoder, so an H.265 main stream will not play in the Frigate UI (live view *or* recordings)
> from a Fedora desktop. On the RLC-812A the codec is **bound to resolution**:
>
> | Main resolution | Codec available |
> |---|---|
> | 3840x2160 | H.265 only |
> | 2560x1440 | H.264 |
> | 2304x1296 | H.264 |
>
> The camera web UI does **not** expose a codec dropdown — changing resolution alone will not flip it.
> Set it over the HTTP API instead:
>
> ```
> curl -sk 'https://192.168.100.17/api.cgi?user=admin&password=<REOLINK_PASSWORD>' \
>   -H 'Content-Type: application/json' \
>   -d '[{"cmd":"SetEnc","action":0,"param":{"Enc":{"channel":0,"audio":0,
>     "mainStream":{"size":"2560*1440","width":2560,"height":1440,"vType":"h264",
>                   "frameRate":25,"bitRate":8192,"profile":"High","gop":2}}}}]'
> ```
>
> Query the supported combinations first with
> `[{"cmd":"GetEnc","action":1,"param":{"channel":0}}]` — the `range` key lists every
> resolution/codec pair the firmware allows. Current settings: `?cmd=GetEnc`.
>
> The camera drops its API for ~30s while the encoder re-initialises. **Restart the Frigate
> deployment afterwards** (`kubectl rollout restart deploy/frigate -n frigate`) — go2rtc caches
> the stream SDP and will keep reporting the old codec until it re-probes.
>
> No firmware fix exists: `v3.1.0.920_2402207844` is the latest build for this hardware, and 4K
> H.264 exceeds the SoC encoder budget. Recordings made **before 2026-07-25** are still HEVC and
> will not play in Chrome.

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
| Face recognition | Frigate 0.17 native (`min_area: 50`, small model). Enabled on cam17_reolink, cam21, cam22. |
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

| Frigate name | Camera | Detect | Record | Frigate native face_rec |
|---|---|---|---|---|
| cam01_front_yard | Xiongmai .11 | Sub 640x480 @ 5fps | Main 720p @ 25fps | No |
| cam02 | Xiongmai .12 | Sub 640x480 @ 5fps | Main 720p @ 25fps | No |
| cam03 | Xiongmai .13 | Sub 640x480 @ 5fps | Main 720p @ 25fps | No |
| cam04–06 | Xiongmai .14–.16 | — | — | No |
| cam17_reolink | Reolink .17 | **`cam17_detect` transcode → 1280x720 @ 10fps** | Main 2560x1440 H.264 @ 25fps | **Yes** |
| cam08_yicam | Yi .19 | Sub 640x480 @ 5fps | Main 720p @ 20fps | No |
| cam22 | ANNKE C500 .22 (Attic) | Main 1920x1080 @ 5fps (TCP) | Main 5MP H.264 @ ? fps | **Yes** |
| cam21 | ANNKE C500 .21 (Bench-deployed 2026-05-10) | Main 1920x1080 @ 5fps (TCP) | Main 5MP H.264 @ ? fps | **Yes** |

> **Face recognition path (Frigate 0.17 native, since 2026-06):** person event -> Frigate's
> built-in recogniser -> `sub_label` written on the event. No external service, no MQTT hop, no
> separate enrolment UI. Manage the face library in the Frigate UI (Settings -> Face Library) or
> over `GET/POST /api/faces`.
>
> Enrolled as of 2026-07-25: `goran` (64 images), `virginia` (79), `miraya` (72), `ivalin` (45),
> `rayna` (32), `Isko Maistora` (9), plus Frigate's own `train` bucket (200).
>
> **Retired 2026-06:** CompreFace (`k3face`, was at `http://192.168.100.212/`) and Double Take,
> along with `/mnt/frigate/double-take/matches/`. All gone from the cluster — no deployment, no
> namespace, `.212` does not answer. Nothing points at them any more.

### Event filter helpers (on k3master, ~/bin/)

Four CLI scripts on k3master (in PATH after `~/bin:` prepend) for browsing Frigate events by
face recognition status. They query Frigate's own `/api/events` and were **not** affected by the
Double Take retirement — only their wording referenced DT.

| Command | What it does |
|---------|--------------|
| frigate-strangers [limit] | Lists recent events with no sub_label (no known face matched — likely strangers). |
| frigate-subject <name> [limit] | Lists events matched to the named subject. **Name is case-sensitive and the library is lowercase** — `frigate-subject ivalin`, not `Ivalin`. |
| frigate-sublabels [limit] | Distribution count of all sub_labels across recent events (sanity check that recognition is firing). Unmatched events are bucketed as `<stranger>`. |
| frigate-family [limit] | Lists events recognized as any family member (`goran,ivalin,miraya,rayna,virginia`). Returned nothing until 2026-07-25 — it hardcoded the DT-era capitalised names while the native face library enrolled them lowercase, and `sub_labels` matching is case-sensitive. `SUBJECTS` lowercased 2026-07-25. |

### Bookmark URLs (Frigate UI sort)

- Family events: `http://192.168.100.209:5000/events?sub_labels=goran,ivalin,miraya,rayna,virginia`
- All events (UI default): `http://192.168.100.209:5000/events`
- Strangers (no sub_label): UI doesn't support negation — use `frigate-strangers` CLI instead

For per-subject filtering append e.g. `?sub_labels=ivalin`. Subject names are **case-sensitive** and the enrolled library is lowercase.

Sub_labels are populated by Frigate's native face recognition when it matches an enrolled face. Strangers and unknown faces get no sub_label, so `frigate-strangers` is the natural feed for browsing non-family clips. The Frigate UI search box also accepts `sub_labels:ivalin` or similar.

> **cam17 note:** Records the main stream (2560x1440 H.264); detection runs on the
> `cam17_detect` go2rtc transcode downscaled to 720p — preserves pixel detail for face
> recognition. Sub stream (640x360) is too low resolution to detect on. Still the most
> expensive camera in the cluster, but cheaper since 2026-07-25: the source is H.264, so
> both the record copy and the detect transcode skip HEVC decode.

### Face Recognition Lessons Learned

1. **Upscaling doesn't help.** Setting detect to 1280x720 on a 640x360 sub stream just stretches pixels — no new detail.
2. **Downscaling preserves detail.** 4K main → 720p detect retains sharp facial features.
3. **go2rtc transcode is CPU-heavy.** `ffmpeg:cam07#video=h264#width=1280#height=720` caused 80-90% CPU (measured while the source was 4K H.265, no `#hardware` flag). The current `cam17_detect` uses `#hardware` on an H.264 source, which is far cheaper.
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
