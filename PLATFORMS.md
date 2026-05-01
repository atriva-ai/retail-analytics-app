# Platform Build & Testing Guide

Cross-platform retail analytics app supporting three hardware targets. Each platform
uses the same shared frontend and backend. Only the edge services differ.

---

## Platform Architecture

| Platform | Compose file | AI service | Video service | Port |
|---|---|---|---|---|
| Intel OpenVINO (x86_64) | `docker-compose.openvino.yml` | `ai_inference` (OpenVINO) | `video_pipeline` (FFmpeg + VAAPI) | 8001 / 8002 |
| Rockchip RK3588 (ARM64) | `docker-compose.rk3588.yml` | `ai_inference` (RKNN NPU) | `video_pipeline` (FFmpeg + rkmpp) | 8001 / 8002 |
| NVIDIA Jetson (ARM64 + CUDA) | `docker-compose.jetson.yml` | `deepstream_service` (TensorRT) | — (DeepStream owns RTSP decode) | 8001 only |

On Jetson, `deepstream_service` replaces **both** `ai_inference` and `video_pipeline`.
It exposes a `/api/v1/video-pipeline/*` shim so the backend needs no platform-specific code.

---

## Prerequisites

### All platforms

```bash
cd retail-analytics-app
git submodule update --init --recursive
cp backend/.env.example backend/.env   # fill in POSTGRES_* values
```

### OpenVINO (x86_64)

- Docker Engine 24+, Docker Compose v2
- Intel GPU optional — `/dev/dri` device exposed for VAAPI/OpenVINO GPU plugin
- No driver install needed; OpenVINO CPU plugin is the fallback

```bash
# Verify Intel GPU (optional)
ls /dev/dri/render*   # renderD128 etc.
```

### RK3588 (ARM64)

**RKNN runtime files must be present before building** (not in git — binary blobs):

```
services/ai-inference-rk3588/rknpu/
  librknnrt.so          # from rknn-toolkit2 release
  rknn_server           # from rknn-toolkit2 release
  rknn_toolkit2-*.whl   # Python wheel from rknn-toolkit2 release
  start_rknn.sh
  restart_rknn.sh
```

Download from the [rknn-toolkit2 releases](https://github.com/rockchip-linux/rknn-toolkit2/releases)
and place in that directory.

**Cross-building from x86 host** (to produce ARM64 images):

```bash
docker buildx create --use
docker buildx inspect --bootstrap
```

**On device** (native ARM64 build, simpler):

```bash
docker compose -f docker-compose.rk3588.yml up --build
```

### Jetson (JetPack 5.x, DeepStream 6.2)

- `nvidia-container-toolkit` installed on host
- Docker daemon configured with nvidia runtime (`/etc/docker/daemon.json`):

```json
{
  "runtimes": {
    "nvidia": { "path": "nvidia-container-runtime", "runtimeArgs": [] }
  },
  "default-runtime": "nvidia"
}
```

- Restart Docker: `sudo systemctl restart docker`

**PeopleNet + ReID ONNX model files** (one-time download, gitignored):

```bash
cd services/deepstream-jetson
# Requires NGC_API_KEY — get from https://ngc.nvidia.com
export NGC_API_KEY=<your_key>
./scripts/download_peoplenet.sh
./scripts/download_reid.sh
```

Models land in `core_engine/models/` and are mounted via the `deepstream_engine_cache`
named volume. TRT engine files are auto-generated on first container start (~5 min on
Jetson Orin NX) and cached — subsequent starts are fast.

**No manual C++ build required.** The Dockerfile compiles the C++ core engine
inside the container during `docker build`. The base image (`nvcr.io/nvidia/deepstream:6.2-triton-multiarch`)
provides all DeepStream headers, GStreamer, and CUDA dependencies.

---

## Service-Level Testing

Test individual services before bringing up the full stack.
These commands assume the service is running on `localhost`.

### OpenVINO — ai_inference

```bash
cd services/ai-inference
docker build -t ai-inference-ov-test .

# Without Intel GPU:
docker run --rm -p 8001:8001 ai-inference-ov-test

# With Intel GPU:
docker run --rm -p 8001:8001 --device /dev/dri:/dev/dri ai-inference-ov-test
```

Health checks:

```bash
curl http://localhost:8001/                            # root
curl http://localhost:8001/model/info                  # available models
curl http://localhost:8001/models                       # model list
curl http://localhost:8001/shared/cameras              # empty list (no video yet)
curl http://localhost:8001/inference/continuous/status?camera_id=1
```

### OpenVINO — video_pipeline

```bash
cd services/video-pipeline
docker build -t video-pipeline-test .
docker run --rm -p 8002:8002 --device /dev/dri:/dev/dri video-pipeline-test
```

Health checks:

```bash
curl http://localhost:8002/api/v1/video-pipeline/health/
curl http://localhost:8002/api/v1/video-pipeline/debug/
```

### RK3588 — ai_inference

```bash
cd services/ai-inference-rk3588

# Cross-build from x86 (image only, no run):
docker buildx build --platform linux/arm64 -t ai-inference-rk3588-test .

# On ARM64 device:
docker build -t ai-inference-rk3588-test .
docker run --rm -p 8001:8001 \
  --device /dev/dri:/dev/dri \
  --device /dev/dma_heap:/dev/dma_heap \
  --group-add video --group-add render \
  ai-inference-rk3588-test
```

Health checks (same API surface as OpenVINO):

```bash
curl http://localhost:8001/model/info
curl http://localhost:8001/shared/cameras
```

### RK3588 — video_pipeline

```bash
cd services/video-pipeline-ffmpeg-rk3588

# On ARM64 device:
docker build -t video-pipeline-rk3588-test .
docker run --rm -p 8002:8002 \
  --device /dev/dri:/dev/dri \
  --device /dev/dma_heap:/dev/dma_heap \
  --group-add video --group-add render \
  video-pipeline-rk3588-test
```

Health checks:

```bash
curl http://localhost:8002/api/v1/video-pipeline/health/
```

### Jetson — deepstream_service

```bash
cd services/deepstream-jetson

# Build (compiles C++ core engine automatically — takes a few minutes):
docker build -t deepstream-retail-test .

# Run (Jetson Orin/Xavier with nvidia runtime):
docker run --rm --runtime=nvidia \
  -p 8001:8001 \
  -e NVIDIA_VISIBLE_DEVICES=all \
  -e NVIDIA_DRIVER_CAPABILITIES=video,compute,utility \
  -e CORE_ENGINE_DIR=/workspace/core_engine \
  -e ZMQ_ENDPOINT=tcp://localhost:5555 \
  deepstream-retail-test
```

Health checks (no cameras registered yet — pipeline deferred until first camera):

```bash
curl http://localhost:8001/                              # service root
curl http://localhost:8001/model/info                    # {architecture: deepstream, ...}
curl http://localhost:8001/models                        # available models
curl http://localhost:8001/cameras                       # {"cameras": []}
curl http://localhost:8001/api/v1/video-pipeline/health/ # {"status": "ok"}
curl http://localhost:8001/api/v1/video-pipeline/debug/  # pipeline state + camera list
curl http://localhost:8001/inference/continuous/status?camera_id=1  # running: false
```

Register a camera and verify pipeline starts:

```bash
curl -X POST "http://localhost:8001/cameras/register?camera_id=1&rtsp_url=rtsp://your-cam/stream"
# Pipeline launches, TRT engines build on first run (~5 min)

curl http://localhost:8001/inference/continuous/status?camera_id=1
# → {"running": true, ...}

curl "http://localhost:8001/shared/cameras/1/detections/latest"
# → {camera_id, detections: [...], detection_count: N}
```

---

## App-Level Testing (Full Stack)

### OpenVINO

```bash
docker compose -f docker-compose.openvino.yml up --build
```

Verify all services are up:

```bash
# Backend
curl http://localhost:8000/api/v1/ai-inference/health/
curl http://localhost:8000/api/v1/ai-inference/model/info/

# AI inference (via backend proxy or direct)
curl http://localhost:8001/model/info

# Video pipeline
curl http://localhost:8002/api/v1/video-pipeline/health/

# Nginx / frontend
curl http://localhost/
open http://localhost:3000
```

Golden path:

1. Open `http://localhost:3000`
2. Add a camera with an RTSP URL — backend calls video pipeline to start decode
3. Enable person detection — backend calls ai_inference to start continuous inference
4. Configure an entrance/exit line — entrance polling thread starts, events appear in DB

### RK3588

**On device (native build):**

```bash
docker compose -f docker-compose.rk3588.yml up --build
```

**Cross-build from x86 host and deploy to device:**

```bash
# On x86: build images
docker buildx bake -f docker-compose.rk3588.yml --set "*.platform=linux/arm64"

# Push to registry or export, then on device:
docker compose -f docker-compose.rk3588.yml up
```

Health check sequence same as OpenVINO.

Note: `force_format=rkmpp` is the hardware decode format for Rockchip — it's passed
by the backend when starting video decode and handled by the FFmpeg rkmpp pipeline service.

### Jetson

```bash
docker compose -f docker-compose.jetson.yml up --build
```

**First run takes ~5 minutes** while TRT engines are generated for PeopleNet and ReID.
Engine files are cached in the `deepstream_engine_cache` volume — subsequent starts are fast.

Verify all services:

```bash
# deepstream_service (serves both AI and video-pipeline APIs)
curl http://localhost:8001/
curl http://localhost:8001/model/info
curl http://localhost:8001/api/v1/video-pipeline/health/

# Backend (uses deepstream_service for all edge calls)
curl http://localhost:8000/api/v1/ai-inference/health/

# Frontend
open http://localhost:3000
```

Golden path (same UI flow as other platforms):

1. Open `http://localhost:3000`
2. Add a camera with an RTSP URL
3. Toggle camera active — backend calls `/api/v1/video-pipeline/decode/` on deepstream_service,
   which registers the RTSP URL and starts the DeepStream GStreamer pipeline
4. Backend checks decode status (`frame_count > 0`) — deepstream shim returns `frame_count=1`
   when pipeline is alive
5. Backend calls `/inference/continuous/start` — no-op (pipeline already running)
6. Backend entrance/exit polling thread calls `/shared/cameras/{id}/detections/latest` every 1s
7. Configure entrance/exit line → events fire when person crosses line

---

## Common Health Check Script

Run this against any platform after `docker compose up`:

```bash
#!/bin/bash
BASE=${1:-http://localhost}

echo "=== Frontend ==="
curl -sI $BASE/ | head -1

echo "=== Backend ==="
curl -s $BASE:8000/api/v1/ai-inference/health/ | python3 -m json.tool 2>/dev/null || echo "unavailable"

echo "=== AI / DeepStream service ==="
curl -s http://localhost:8001/model/info | python3 -m json.tool

echo "=== Video pipeline (or DeepStream shim) ==="
curl -s http://localhost:8001/api/v1/video-pipeline/health/ || \
curl -s http://localhost:8002/api/v1/video-pipeline/health/ | python3 -m json.tool
```

---

## Troubleshooting

### OpenVINO: "no Intel GPU found"

The GPU plugin falls back to CPU automatically. Remove `--device /dev/dri` if the host
has no Intel GPU and the container logs show VAAPI errors.

### RK3588: "librknnrt.so not found"

The RKNN runtime .so was not placed in `services/ai-inference-rk3588/rknpu/` before build.
See Prerequisites above.

### RK3588: cross-build takes too long

Use `--cache-from` with a registry or build natively on the device. The RK3588 pip install
layer (especially numpy) is slow under QEMU emulation.

### Jetson: TRT engine build fails or hangs

- Ensure ONNX model files exist in `core_engine/models/peoplenet/` and `core_engine/models/reid/`
  before `docker compose up`.
- `deepstream_engine_cache` volume persists engines across restarts — if the volume is corrupt,
  remove it: `docker volume rm retail-analytics-app_deepstream_engine_cache`

### Jetson: pipeline not starting after camera register

Check deepstream_service logs:

```bash
docker compose -f docker-compose.jetson.yml logs deepstream_service
```

Look for:
- `"core_engine binary not found"` — C++ build failed during `docker build`
- `"Launched core_engine PID"` — binary started successfully
- `"ZMQ subscriber connected"` — detection stream is active

### Jetson: detections not arriving

Verify ZMQ endpoint matches between C++ publisher and Python subscriber.
Both must use `tcp://localhost:5555` within the same container.
The `ZMQ_ENDPOINT` env var controls the Python subscriber endpoint;
the C++ `default.yaml` `output.endpoint` controls the publisher bind address.
