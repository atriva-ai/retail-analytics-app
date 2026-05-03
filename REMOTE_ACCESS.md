# Live Video Streaming & Remote Access Architecture

Analysis of streaming strategies for moving from snapshot polling to live video,
across all supported hardware platforms (Jetson, CPU, RK3588, OpenVINO), with
support for remote access through a web relay.

---

## Core Design Principle: Separate Video from Metadata

The most important architectural decision is **never re-encode video on the edge device
for the purpose of burning in bounding boxes**. Instead:

```
Camera (H264 RTSP)
    │
    │  H264 bitstream — zero decode, zero re-encode
    ▼
MediaMTX  ──transmux──▶  HLS segments  ──HTTP──▶  Browser <video>
                                                          │
                                                    <canvas> overlay
                                                          ▲
                                                  WebSocket bbox events
                                                   {"detections": [...]}
```

**Why this matters on Jetson specifically:**
Jetson Orin NX has one hardware video encoder (NVENC). Running `nvv4l2h264enc` for an
annotated output stream consumes that encoder for display, capping the number of inference
channels. With pass-through (transmux), the encoder is not used at all — only the NVDEC
decoder and GPU inference engines are active. The OSD chain (`nvv4l2h264enc` + `filesink`
or `nveglglessink`) should be **disabled in production** and kept only for development/debug.

Frontend rendering of bboxes via `<canvas>` overlay is flexible, zero-cost on the server,
and lets the UI control what's shown (labels, track IDs, confidence, zone polygons) without
any server-side re-encoding.

**This is the pattern used by modern analytics platforms** (Axis ACAP, Milestone XProtect,
Genetec, NVIDIA Metropolis) — raw stream + metadata sidecar.

---

## Current State vs. Target

| Capability | Current | Target |
|---|---|---|
| Camera preview in UI | Snapshot JPEG, 5s poll | Live HLS video in `<video>` |
| Detection overlay | None | `<canvas>` overlay on video, fed by WS |
| Remote access | Works (REST + WS over HTTP) | Works (HLS is HTTP, same path) |
| Jetson encoding for display | OSD chain (dev/debug only) | Disabled in production |
| LAN latency | ~5s (snapshot) | <2s (HLS) or <200ms (WebRTC WHEP) |
| Remote latency | ~5s (snapshot) | 3–6s (HLS standard) or 1–2s (LL-HLS) |

---

## Protocol Comparison

### HLS — HTTP Live Streaming

**How it works:** Server segments H264/H265 into 1–6s `.ts` or `.fmp4` files plus an `.m3u8`
playlist. Browser downloads segments sequentially via normal HTTP GET requests.

**Latency:** 3–6s standard; 1–2s with Low-Latency HLS (LL-HLS, IETF draft).

**Remote relay:** ✅ Ideal — segments are plain HTTP responses, work through any HTTP
proxy, CDN, or tunnel. No UDP, no persistent connections (other than normal keep-alive).

**Browser support:** Native in Safari. Chrome/Firefox need hls.js (42 KB minified).

**On-device cost for transmux:** Near-zero — H264 NAL units from camera are just
repackaged into MPEG-TS containers. No decode, no encode.

**References:**
- RFC 8216 — HTTP Live Streaming: https://datatracker.ietf.org/doc/html/rfc8216
- Apple LL-HLS spec: https://developer.apple.com/documentation/http-live-streaming/protocol-extension-for-low-latency-hls
- hls.js: https://github.com/video-dev/hls.js

---

### WebRTC WHEP — WebRTC HTTP Egress Protocol

**How it works:** Browser initiates an ICE/DTLS/SRTP session with the server via an HTTP
signalling endpoint (WHEP). Video travels over UDP (SRTP). Sub-second latency due to no
buffering.

**Latency:** <200ms glass-to-glass on LAN.

**Remote relay:** ⚠️ Problematic. WebRTC uses UDP which NAT traversal requires STUN/TURN
infrastructure. A pure HTTP relay cannot forward UDP. Needs either:
- TURN server (adds cost and bandwidth relay), or
- Cloudflare TURN / Metered TURN, or
- Falls back to TCP over TURN (latency penalty, defeats the purpose)

**Best use:** LAN-only monitoring where sub-second latency matters (live security watch,
real-time intervention). Offer alongside HLS as an auto-detected fast path on the same
MediaMTX service.

**References:**
- IETF WHEP draft: https://datatracker.ietf.org/doc/draft-ietf-wish-whep/
- WebRTC STUN/TURN overview: https://webrtc.org/getting-started/turn-server
- Metered TURN (free tier): https://www.metered.ca/tools/openrelay/

---

### MJPEG — Motion JPEG over HTTP

**How it works:** Server sends an endless `multipart/x-mixed-replace` HTTP response,
each part is a JPEG frame. Browser `<img>` tag renders each frame as it arrives.

**Latency:** ~100–300ms.

**Remote relay:** ✅ It is a regular HTTP response body — works through any proxy.

**Drawbacks:** No temporal compression (each frame is independent JPEG). Bandwidth is
5–15× higher than H264 for the same quality. Typically 5–15 fps for 720p on typical
uplinks. Acceptable as a **fallback** when HLS is not available or for very low-bandwidth
connections where H264 decoder overhead on the device matters.

**References:**
- RFC 2046 §5.1 — Multipart media types: https://datatracker.ietf.org/doc/html/rfc2046#section-5.1
- OpenCV VideoWriter MJPEG: https://docs.opencv.org/4.x/dd/d9e/classcv_1_1VideoWriter.html

---

### MSE + WebSocket — Media Source Extensions with chunked H264

**How it works:** Backend sends H264 chunks (fMP4 or Annex B with timestamps) over a
WebSocket. Frontend uses the MSE API to feed chunks into a `SourceBuffer` attached to
a `<video>` element.

**Latency:** 1–2s with small chunks (200ms GOP).

**Remote relay:** ✅ WebSocket over HTTPS works through any HTTP/1.1 relay.

**Drawbacks:** Highest implementation complexity. Safari MSE support is incomplete.
Requires careful timestamping and buffer management. MediaMTX + HLS achieves similar
latency with far less code.

**References:**
- W3C MSE spec: https://www.w3.org/TR/media-source/
- Realtime video with MSE: https://web.dev/articles/media-mse-basics

---

## Recommended Stack: MediaMTX

MediaMTX is a single-binary media server that accepts RTSP/RTMP/SRT/WebRTC and
simultaneously outputs HLS, DASH, WebRTC WHEP, RTMP, and SRT — all from the same
input source.

**Why MediaMTX over nginx-rtmp or GStreamer HLS:**
- No transcoding in default mode — pure transmux (H264 pass-through)
- REST API for dynamic path management (add/remove camera with one HTTP call)
- Outputs HLS and WebRTC WHEP from the same source simultaneously
- Actively maintained, production-proven in surveillance deployments
- Docker image available; minimal configuration

**References:**
- MediaMTX GitHub: https://github.com/bluenviron/mediamtx
- MediaMTX path API: https://github.com/bluenviron/mediamtx/blob/main/docs/api.md
- Benchmark (multi-stream transmux overhead): https://github.com/bluenviron/mediamtx#performance

---

## Architecture for Each Platform

### All platforms — raw stream path (production)

```
RTSP camera (H264)
    │
    │  H264 NAL units — no decode
    ▼
MediaMTX  (transmux to HLS segments)
    │
    │  HTTP /streams/{camera_id}/index.m3u8
    ▼
nginx  →  Browser <video> (hls.js)
              +
           <canvas> overlay  ←  WebSocket /ws/cameras/{id}/live
                                 {"detections": [{"bbox":[x1,y1,x2,y2], "track_id":3, ...}]}
```

No re-encoding. No platform-specific code in the streaming path.

### Jetson / DeepStream — production configuration

- **OSD chain disabled**: Remove `nvmultistreamtiler`, `nvdsosd`, `nvv4l2h264enc`, `filesink`
  from the GStreamer pipeline in production configs. These are useful for development
  (see what the AI is detecting on-device) but consume NVENC capacity and add ~50ms latency.
- **Raw RTSP passed to MediaMTX directly** — MediaMTX proxies the original camera RTSP URL.
  DeepStream decodes the same stream independently via NVDEC for inference.
- **Bbox metadata** reaches the frontend via the existing WebSocket broadcast from the
  analytics processor. The frontend `<canvas>` overlay renders boxes on top of the HLS stream.

```
Camera RTSP
    ├──▶ DeepStream (NVDEC → PeopleNet → NvDCF → ReID → ZMQ)
    │                                                     │
    │                                           Backend analytics processor
    │                                                     │
    │                                           WebSocket /ws/cameras/{id}/live
    │                                           {"detections": [{bbox, track_id, global_id}]}
    │                                                     │
    └──▶ MediaMTX (transmux)                             ▼
              │                              Browser <canvas> overlay
              ▼
         HLS /streams/{id}/
              │
              ▼
         Browser <video>
```

Both DeepStream and MediaMTX connect to the camera RTSP source independently.
The camera is the only H264 encoder in this path.

### Standard CPU / OpenVINO / RK3588

Same architecture. MediaMTX proxies RTSP directly. AI inference services continue
to poll detections via the existing REST path. No streaming code changes in inference services.

---

## Bbox WebSocket Message Extension

The current WebSocket detection message (`count` only) needs to include bbox data for
frontend canvas rendering:

```json
{
  "type": "detections",
  "camera_id": 1,
  "count": 3,
  "ts": 1713000000,
  "detections": [
    {
      "track_id": 7,
      "global_id": 2,
      "class_name": "person",
      "confidence": 0.91,
      "bbox_norm": [0.12, 0.08, 0.31, 0.74]
    }
  ]
}
```

`bbox_norm` is `[x1, y1, x2, y2]` in normalised `[0,1]` coordinates (already computed
by `DetectionAdapter`). The frontend scales to canvas pixel space:
```typescript
const x1 = bbox_norm[0] * canvas.width
const y1 = bbox_norm[1] * canvas.height
// ...
ctx.strokeRect(x1, y1, (bbox_norm[2]-bbox_norm[0])*canvas.width, ...)
```

Timestamp-based sync between video and metadata: retail analytics does not require
frame-perfect sync. A simple "draw latest received bboxes" approach (best-effort, ~100–500ms
drift acceptable) is standard practice for dwell time / occupancy analytics. Frame-perfect
sync would require embedding timestamps in the H264 SEI or using WebRTC data channels
alongside the video — unnecessary complexity for this use case.

---

## Remote Access Options

### Cloudflare Tunnel (recommended)

Runs a lightweight daemon (`cloudflared`) on the edge device. Creates a persistent
outbound HTTPS tunnel to Cloudflare's edge network. No inbound port forwarding required,
works behind CGNAT and corporate NAT.

- HLS streams proxy transparently (HTTP chunked responses)
- WebSocket connections work (Cloudflare proxies WS over HTTP/1.1 Upgrade)
- Free tier supports unlimited bandwidth through Cloudflare's network
- Custom domain or `*.trycloudflare.com` subdomain

**References:**
- Cloudflare Tunnel docs: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
- cloudflared Docker: https://hub.docker.com/r/cloudflare/cloudflared

### Tailscale Funnel (alternative)

Tailscale's `funnel` command exposes a local HTTPS port to the public internet through
Tailscale's relay network. Simpler for single-device setups.

- Reference: https://tailscale.com/kb/1223/tailscale-funnel

### Self-hosted relay (VPS)

nginx on a VPS with a persistent tunnel from the edge device (SSH reverse tunnel or
WireGuard). Maximum control, requires VPS cost (~$5/mo).

- Reference: https://www.nginx.com/blog/websocket-nginx/ (WS proxy config)

---

## Implementation Sequence (when ready)

1. **Add MediaMTX service** to all docker-compose variants. Configure RTSP→HLS transmux
   with 1s segments (LL-HLS option for sub-2s latency).
2. **Backend on camera activate**: call MediaMTX path API to register RTSP URL.
3. **Nginx**: add `/streams/` proxy location → MediaMTX HLS port.
4. **Extend WS message**: include `detections` array with `bbox_norm` in the analytics
   processor broadcast (zero backend architectural change, just add fields to the dict).
5. **Frontend**: replace `<img>` snapshot with `<video>` (hls.js) + `<canvas>` overlay.
6. **Jetson**: gate OSD chain behind a `ENABLE_OSD=false` env var; default off in
   production compose, on in dev compose.
7. **Optional LAN fast-path**: detect WHEP availability, use WebRTC for <200ms when local.
8. **Optional remote**: add `cloudflared` sidecar to docker-compose with tunnel token env var.
