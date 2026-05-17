---

# Additional Firmware Analysis Findings
bootloader pass #Ux6@9V&4_Rz
## Confirmed Video Configuration

Inspection of extracted configuration files revealed:

### Main Stream

```json
{
  "Compression": "H.265",
  "Resolution": "4K",
  "FPS": 10
}
```

### Extra/Sub Stream

```json
{
  "Compression": "H.265",
  "FPS": 15
}
```

Key implications:
- Both streams are configured as H.265
- Main stream uses 4K at only 10 FPS
- Current configuration is heavily optimized for:
  - storage efficiency
  - surveillance workloads
  - bandwidth reduction
- Current configuration is NOT optimized for:
  - realtime teleoperation
  - low latency
  - responsive operator feedback

Current working hypothesis:
- major latency contribution originates inside the camera firmware itself before the stream even reaches the Radxa pipeline

Likely causes:
- aggressive H.265 buffering
- long GOP intervals
- vendor encoder queueing
- surveillance-oriented tuning defaults
- low FPS operation

---

# Media Stack Findings

## Confirmed XM/Xiongmai Ecosystem

The following files were identified:

```text
LIBXMVIDEO.json
LIBXMVIDEO_Sensors.json
IPC_UNKNOWN_LIBXMVIDEO.json
```

This strongly confirms:
- XM/Xiongmai-derived software stack
- Novatek + XM hybrid firmware ecosystem

---

# Confirmed Novatek Video Pipeline

The firmware contains the following Novatek media modules:

```text
kdrv_h26x.ko
kflow_videoenc.ko
kdrv_videocapture.ko
kflow_videoprocess.ko
```

Additional pipeline references discovered:

```text
nvt_h26x
nvt_jpg
nvt_audio
sensor
ife
ipe
ise
vpe
```

Approximate internal architecture:

```text
sensor
  ↓
IFE  (image front-end)
  ↓
IPE  (image processing)
  ↓
ISE  (scaler/effects)
  ↓
VPE  (video processing engine)
  ↓
H26X encoder
  ↓
RTSP/web stack
```

This suggests:
- hardware capability is likely NOT the main bottleneck
- latency problems are probably software/configuration related

---

# Startup Architecture Findings

## Media Startup Path

Important startup references discovered:

```text
/etc/init.d/rcS
XmServices_Mgr /usr/sbin/AppRun.sh /mnt/mtd/Config/
```

Current understanding:
- `/sbin/media` is mostly diagnostic tooling
- `XmServices_Mgr` + `AppRun.sh` likely orchestrate the real media stack

---

# Current Architectural Decision

## IMPORTANT STRATEGIC PIVOT

The project strategy has evolved significantly.

Initial idea:
- deeply modify camera firmware
- potentially replace media stack
- potentially deploy OpenIPC or custom firmware

Current preferred strategy:
- keep camera firmware as stock and stable as possible
- minimize camera-side modifications
- move ALL complexity and latency optimization into the Radxa platform

Reasoning:
- Radxa already performs:
  - H.265 decode
  - H.264 encode
- Current pipeline already achieves:
  - sub-100ms processing latency on the Radxa
- Radxa is:
  - easier to automate
  - GitHub-managed
  - containerized
  - IaC-friendly
  - AI-assisted
  - easier to recover/update
- Camera firmware hacking:
  - increases operational complexity
  - reduces maintainability
  - creates fragile deployment dependencies

New architectural principle:

```text
Camera must remain replaceable.
Radxa pipeline owns latency behavior.
```

---

# Preferred Operational Architecture

Target architecture:

```text
Stock Camera Firmware
  ↓ RTSP stream
Radxa ingest/normalize layer
  ↓ decode camera stream
Radxa low-latency H.264 encode
  ↓
MediaMTX / WebRTC
  ↓
Browser operator UI
```

Advantages:
- camera becomes mostly interchangeable
- latency optimization centralized
- all tuning/configuration version-controlled
- no dependency on vendor firmware hacks
- easier field deployment
- easier remote maintenance
- easier AI-assisted iteration

---

# Camera-Side Modification Philosophy

## Camera Firmware Changes Should Be Minimal

Avoid:
- bootloader modifications
- custom low-level firmware
- OpenIPC migration (for now)
- deep binary patching
- dependency on UART/TFTP workflows

Prefer:
- normal configuration changes only
- stock firmware retention
- reversible modifications
- web/UI/API configuration where possible

---

# Concrete Camera-Side Tuning Plan

## Goal

Convert streams from:
- H.265 surveillance-oriented profiles

to:
- H.264 low-latency teleoperation-oriented profiles

Without:
- major firmware surgery
- replacing vendor media stack

---

# Planned Stream Configuration

## Extra/Sub Stream Target

Primary target stream for teleoperation:

```json
{
  "VideoEnable": true,
  "AudioEnable": false,
  "Video": {
    "Resolution": "HD1",
    "BitRateControl": "CBR",
    "Compression": "H.264",
    "BitRate": 3072,
    "FPS": 25,
    "GOP": 10,
    "Quality": 4
  }
}
```

Potential future variant:

```json
{
  "Resolution": "D1",
  "Compression": "H.264",
  "FPS": 25,
  "GOP": 5
}
```

---

# Concrete Investigation / Implementation Plan

## Phase 1 — Safe Config Inspection

Goals:
- identify actual encoder config files
- identify stream profile persistence
- identify whether H.264 already supported
- identify whether GOP configurable

Targets:
- `AVEnc.custom`
- XM video JSON files
- startup scripts
- stream config files

---

## Phase 2 — Minimal Camera Reconfiguration

Goals:
- switch substream to H.264
- increase FPS
- reduce GOP interval
- disable audio
- disable smart codec behavior
- lower resolution if needed

Rules:
- no bootloader modifications
- no permanent firmware patching
- no OpenIPC migration yet

---

## Phase 3 — Radxa Pipeline Optimization

This becomes the PRIMARY engineering focus.

Goals:
- standardized ingest pipeline
- low-latency normalization layer
- deterministic H.264 encoding
- centralized latency tuning
- IaC-managed configuration

Likely optimization areas:
- FFmpeg tuning
- queue depth
- encoder presets
- WebRTC buffering
- MediaMTX tuning
- observability tooling
- latency instrumentation

All managed from:
- GitHub
- Docker
- compose files
- automation tooling
- AI-assisted workflows

---

## Phase 4 — Latency Measurement & Validation

Goals:
- measure:
  - camera-side latency
  - decode latency
  - encode latency
  - WebRTC latency
  - browser render latency

Build:
- latency observability tooling
- stacked latency breakdowns
- telemetry dashboards

---

# MJPEG Investigation Status

MJPEG support remains uncertain.

Current understanding:
- JPEG-related components exist inside firmware
- no confirmed exposed MJPEG stream yet

Current project decision:
- MJPEG investigation is lower priority
- H.264 low-latency pipeline preferred first

Reasoning:
- MJPEG bandwidth cost extremely high
- H.264 likely sufficient if buffering reduced properly
- Radxa already capable of efficient low-latency transcoding

---

# Current Strategic Conclusion

The most promising architecture is now considered:

```text
Cheap stock camera
→ Radxa normalization/transcoding layer
→ standardized low-latency H.264/WebRTC pipeline
→ browser operator UI
```

The Radxa is increasingly viewed as:
- the real media platform
- the real latency control layer
- the maintainable engineering surface

The camera should ideally become:
- replaceable
- dumb
- minimally modified
- operationally boring

This significantly reduces:
- firmware maintenance burden
- recovery complexity
- dependency on reverse-engineered vendor internals
- operational fragility in the field
